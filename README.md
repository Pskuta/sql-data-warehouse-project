# SQL Data Warehouse Project

Building a modern data warehouse with SQL Server, including ETL process, data modeling and analytics.

![SQL Server](https://img.shields.io/badge/SQL%20Server-T--SQL-CC2927?logo=microsoftsqlserver&logoColor=white)

## Project Overview

Sales, customer and product data is split between two source systems (CRM and ERP) delivered as CSV files. The files use inconsistent keys, codes and formats, and they contain duplicates, invalid dates and mismatched sales values. This makes direct reporting unreliable.

This project consolidates both sources into a single SQL Server data warehouse. It cleans and standardizes the data and exposes it as a star schema for analytical queries.

**Scope:**
- Ingest CRM and ERP CSV files into a raw layer.
- Cleanse, standardize and integrate the data.
- Publish a business-ready star schema.
- Validate data quality with SQL checks.
- Answer questions about customers, products and sales trends.

## Tech Stack

| Area | Tool |
|------|------|
| Database | SQL Server 17 |
| Language | T-SQL (stored procedures, views, window functions) |
| Ingestion | `BULK INSERT` from CSV |
| Modeling | Medallion architecture, star schema |
| Diagrams | draw.io |
| Client tool | SQL Server Management Studio (SSMS) |
| Version control | Git, GitHub |

## Repository Structure

```text
sql-data-warehouse-project/
├── datasets/            # Source CSV files
│   ├── source_crm/      # CRM files: customers, products, sales
│   └── source_erp/      # ERP files: customer attributes, locations, product categories
├── docs/                # draw.io diagrams 
├── scripts/             # SQL scripts for building and loading the warehouse
│   ├── init_database.sql   # Creates DataWarehouse database and bronze/silver/gold schemas
│   ├── bronze/          # Bronze DDL and BULK INSERT load procedure
│   ├── silver/          # Silver DDL and cleansing/transformation procedure
│   └── gold/            # Gold views (star schema)
├── tests/               # Data quality checks for Silver and Gold
└── README.md
```


## Getting Started

### Prerequisites

- SQL Server 2017 Express
- SQL Server Management Studio (SSMS)
- Read access for the SQL Server service account to the folder with CSV files
- Git

### Installation and run

1. Clone the repository:

   ```bash
   git clone https://github.com/Pskuta/sql-data-warehouse-project.git
   cd sql-data-warehouse-project
   ```

2. Copy the `datasets/` folder to a location readable by SQL Server. The scripts expect `C:\sql\dwh_project\datasets\`. If you use another path, edit the file paths in `scripts/bronze/proc_load_bronze.sql`.
3. Open SSMS and connect to your SQL Server instance.
4. Run `scripts/init_database.sql`. **Warning:** it drops and recreates the `DataWarehouse` database.
5. Run `scripts/bronze/ddl_bronze.sql` to create the Bronze tables.
6. Run `scripts/bronze/proc_load_bronze.sql` to create the procedure, then execute it:

   ```sql
   EXEC bronze.load_bronze;
   ```

7. Run `scripts/silver/ddl_silver.sql` to create the Silver tables.
8. Run `scripts/silver/proc_load_silver_layer.sql`. It creates `silver.load_silver` and executes it at the end of the script.
9. Run `scripts/gold/ddl_gold.sql` to create the Gold views.
10. Run `tests/quality_checks_silver.sql` and `tests/quality_checks_gold.sql` and review the results.
11. Query the Gold layer, for example:

    ```sql
    SELECT TOP 10 * FROM gold.fact_sales;
    ```

## Architecture

The warehouse follows the **Medallion architecture** (Bronze → Silver → Gold) and uses three schemas in the `DataWarehouse` database.

| Layer | Schema | Object type | Purpose | Load method |
|-------|--------|-------------|---------|-------------|
| Bronze | `bronze` | Tables | Raw copy of source CSV files, no transformations | `BULK INSERT` (full load, truncate and insert) |
| Silver | `silver` | Tables | Cleansed, standardized and typed data | Stored procedure `silver.load_silver` (full load) |
| Gold | `gold` | Views | Business-ready star schema | `CREATE VIEW` over Silver |

**Data flow:** `CSV (CRM, ERP)` → `bronze.*` → `silver.*` → `gold.dim_*` / `gold.fact_sales` → SQL analytics / BI tools.


### Diagrams

**Data flow diagram** (source: `docs/Data_flow_diagram.drawio`)
<p align="center">
<img width="689" height="479" alt="Diagram bez tytułu drawio" src="https://github.com/user-attachments/assets/22bb046e-15fb-4c21-bd27-9fb71b73ea51" />
</p>

**Data integration model** (source: `docs/Data integration model.drawio`)
<p align="center">
<img width="962" height="411" alt="Data integration model (1) drawio" src="https://github.com/user-attachments/assets/dddc7ef4-96f4-4bdd-b454-48ea38f280c8" />
</p>

**Data mart (star schema)** (source: `docs/Data mart.drawio`)
<p align="center">
<img width="792" height="538" alt="Data mart (1) drawio" src="https://github.com/user-attachments/assets/43764c35-5b93-48c0-85e2-1ee2039aea3a" />
</p>

## Data Sources and Data Model

### Data sources

| System | File | Content |
|--------|------|---------|
| CRM | `cust_info.csv` | Customer master data |
| CRM | `prd_info.csv` | Product data with validity dates |
| CRM | `sales_details.csv` | Sales order lines |
| ERP | `CUST_AZ12.csv` | Additional customer attributes (birthdate, gender) |
| ERP | `LOC_A101.csv` | Customer country |
| ERP | `PX_CAT_G1V2.csv` | Product categories and subcategories |

### Gold layer: star schema

| Object | Type | Grain / role | Key columns |
|--------|------|--------------|-------------|
| `gold.fact_sales` | Fact | One row per order line | `order_number`, `product_key`, `customer_key`, `order_date`, `shipping_date`, `due_date`, `sales_amount`, `quantity`, `price` |
| `gold.dim_customers` | Dimension | One row per customer | `customer_key` (surrogate), `customer_id`, `customer_number`, `first_name`, `last_name`, `country`, `marital_status`, `gender`, `birthdate`, `create_date` |
| `gold.dim_products` | Dimension | One row per current product | `product_key` (surrogate), `product_id`, `product_number`, `product_name`, `category_id`, `category`, `subcategory`, `maintenance`, `cost`, `product_line`, `start_date` |

**Relationships:**
- `fact_sales.customer_key` → `dim_customers.customer_key`
- `fact_sales.product_key` → `dim_products.product_key`

**Modeling notes:**
- Surrogate keys are generated with `ROW_NUMBER()` inside the views.
- `dim_customers` uses CRM as the primary source for gender and falls back to ERP when CRM has no value.
- `dim_products` keeps only current records (`prd_end_dt IS NULL`), so historical product versions are excluded.

## ETL Process

### 1. Extract
CSV files from `datasets/source_crm` and `datasets/source_erp` are the source. No source system connection is used.

### 2. Load to Bronze
`EXEC bronze.load_bronze;` truncates each Bronze table and loads the matching CSV with `BULK INSERT` (`FIRSTROW = 2`, `FIELDTERMINATOR = ','`, `TABLOCK`). It prints per-table and total load durations.

### 3. Transform and load to Silver
`EXEC silver.load_silver;` truncates the Silver tables and inserts cleansed data from Bronze. Main rules:

| Table | Rule |
|-------|------|
| `crm_cust_info` | Remove rows with NULL `cst_id`. Keep only the latest record per customer (`ROW_NUMBER()` by `cst_create_date DESC`). Trim names. Map marital status (`S`/`M`) and gender (`M`/`F`) to readable values, with `n/a` as default. |
| `crm_prd_info` | Derive `cat_id` and `prd_key` from the composite product key. Replace NULL cost with 0. Map product line codes (`M`, `R`, `S`, `T`) to names. Compute `prd_end_dt` as the day before the next `prd_start_dt` (`LEAD`). |
| `crm_sales_details` | Convert integer dates (`YYYYMMDD`) to `DATE`. Invalid values (0 or wrong length) become NULL. Recalculate `sales` as `quantity * ABS(price)` when it is missing, non-positive or inconsistent. Derive `price` from `sales / quantity` when it is missing or non-positive. |
| `erp_cust_az12` | Remove the `NAS` prefix from `cid`. Set future birthdates to NULL. Standardize gender values. |
| `erp_loc_a101` | Remove `-` from `cid`. Normalize country codes (`DE` → Germany, `US`/`USA` → United States). Replace empty values with `n/a`. |
| `erp_px_cat_g1v2` | Loaded as is. |

### 4. Publish Gold
`scripts/gold/ddl_gold.sql` (re)creates the dimension and fact views, joining CRM and ERP data on the cleansed keys.

### 5. Validate
The scripts in `tests/` check data quality. Checks marked "Expectation: No Results" in the scripts should return no rows.
- **Silver** (`quality_checks_silver.sql`):
  - NULL and duplicate primary keys (customers, products).
  - Unwanted spaces in string fields.
  - Negative or NULL product cost.
  - Invalid dates and date order (product start/end, order/ship/due).
  - Consistency of `sales = quantity * price`.
  - Birthdate range.
  - Standardized values (marital status, product line, gender, country, maintenance).
- **Gold** (`quality_checks_gold.sql`): uniqueness of surrogate keys in both dimensions and referential integrity between the fact and dimensions.

## Analytics & Insights

The model supports questions such as:
- Which countries and customer segments generate the most revenue?
- Which products and categories drive sales?
- How do sales change over time?

**Revenue by country:**

```sql
SELECT
    c.country,
    COUNT(DISTINCT c.customer_key) AS customers,
    SUM(f.sales_amount)            AS total_sales
FROM gold.fact_sales f
LEFT JOIN gold.dim_customers c ON c.customer_key = f.customer_key
GROUP BY c.country
ORDER BY total_sales DESC;
```
<p align="center">
<img width="254" height="155" alt="image" src="https://github.com/user-attachments/assets/e8504c07-0db4-420f-81b0-3435d7edaa77" />
</p>

**Top 10 products by revenue:**

```sql
SELECT TOP 10
    p.product_name,
    p.category,
    SUM(f.sales_amount) AS total_sales,
    SUM(f.quantity)     AS total_quantity
FROM gold.fact_sales f
LEFT JOIN gold.dim_products p ON p.product_key = f.product_key
GROUP BY p.product_name, p.category
ORDER BY total_sales DESC;
```
<p align="center">
<img width="367" height="211" alt="image" src="https://github.com/user-attachments/assets/76329ecf-6e30-4c65-abf3-2d4e0bbb9cd3" />
</p>

**Monthly sales trend:**

```sql
SELECT
    DATEFROMPARTS(YEAR(f.order_date), MONTH(f.order_date), 1) AS order_month,
    SUM(f.sales_amount)                                       AS total_sales,
    COUNT(DISTINCT f.order_number)                            AS orders
FROM gold.fact_sales f
WHERE f.order_date IS NOT NULL
GROUP BY DATEFROMPARTS(YEAR(f.order_date), MONTH(f.order_date), 1)
ORDER BY order_month;
```
<p align="center">
<img width="221" height="741" alt="image" src="https://github.com/user-attachments/assets/d952d0d0-a52a-4bb9-b46b-0d722b10aa82" />
</p>
