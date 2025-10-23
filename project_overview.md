# Project Overview

## Business Context

The objective of this project was to design and implement a complete ETL pipeline and Data Warehouse for retail sales data using KNIME Analytics Platform, PostgreSQL, and AWS infrastructure.
The system enables structured reporting, incremental data updates, and scalable architecture following the star schema approach.

The project represents a realistic enterprise workflow where sales transactions — including product, payment method, and loyalty card information — are collected, transformed, and loaded into a centralised warehouse for analytical use.

---

## Cloud Infrastructure and Setup

All database operations were performed on an AWS-hosted PostgreSQL instance, allowing remote and scalable data handling.
- Cloud Provider: Amazon Web Services (AWS)
- Access Tool: pgAdmin client for database management and testing
- Connection: KNIME’s native PostgreSQL Connector node configured to connect directly to the AWS-hosted database

This setup allowed full control over database configuration and made it possible to replicate production-like ETL workflows from the local KNIME environment.

---

## Data Source

The data originates from transactional CSV files containing sales records:
- fact_sales_1.csv – used for the initial (full) load
- fact_sales_2.csv – used to simulate a delta (incremental) load

These datasets represent retail transactions, each with information about the product, price, quantity, payment type, loyalty card usage, and credit card identifier.

---

## Database Architecture

The warehouse is divided into two layers — Staging and Core — designed following a dimensional star schema.

**Fact Table — core.sales**

The sales fact table contains the transactional data enriched with foreign keys and calculated measures.

Design decisions:
- Primary Key: Natural key (transaction_id) — simple and unique.
- Delta Column: transactional_date, used for incremental load detection.
- Foreign Keys:
    - product_fk → core.dim_product
    - payment_fk → core.dim_payment
    - transactional_date_fk → core.dim_date (planned)
    - customer_id → reserved for future customer dimension
- Degenerate Dimension: credit_card (maintained as-is).
- Measures: cost, price, quantity
- Derived:
    - total_price = price × quantity
    - total_cost = cost × quantity
    - profit = total_price − total_cost

**Dimension Tables**

1. Product Dimension — core.dim_product

    Source: staging.dim_product

    Columns:
    - product_pk (surrogate key)
    - product_id (natural key)
    - product_name
    - brand
    - category
    - subcategory

    Transformations:
    - Splitting brand from product_name
    - Removing extra spaces/tabs and redundant parentheses
    - Converting inconsistent strings

2. Payment Dimension — core.dim_payment

    Defined as a junk dimension combining:
    - payment (visa, mastercard, americanexpress, cash)
    - loyalty_card (T/F)

    Key logic:
    - Replaced null payments with 'cash' via:

    COALESCE(payment, 'cash')
    - Generated 6 unique combinations (4×2)
    - Created auto-incremented payment_pk

3. Date Dimension — core.dim_date (planned)

    Not fully implemented in this phase but referenced via transactional_date_fk for future time-based analytics.

---

## ETL Process (KNIME)

**Staging Layer**
1. Initial Load
    - Imported fact_sales_1.csv into staging.sales using pgAdmin (**write_the_table_staging_dim_product** workflow).
2. Transformation Workflows
    - Fixed inconsistent formatting and applied brand extraction logic (**load_and_transform_staging_dim_product** workflow).
3. Data Cleaning
    - Handled nulls, casted numeric types (e.g., cost) after refreshing rows based on **transactional_date** column (**get_last_load_staging_sales** workflow).
4. Output
    - Clean data written to staging tables ready for core layer transfer (**get_last_load_staging_sales**) workflow.



**Core Layer**
1. Product Dimension (**core_dim_product** workflow) 
    - Loaded cleaned staging data into core.dim_product.
2. Payment Dimension (**core_dim_payment** workflow)
    - Extracted unique combinations of payment × loyalty_card.
    - Replaced nulls and generated surrogate keys.
3. Sales Fact (**core_sales_fact** workflow)
    - Combined all relevant dimensions and transactional data.
    - Used KNIME Math Formula nodes to compute:
        - total_price
        - total_cost
        - profit
    - Wrote the results into the core sales fact table.

---

## Incremental (Delta) Load Logic

Implemented a mechanism to refresh only new rows based on the transactional_date column (**set_last_load_sales** workflow).

**Full Load (first execution):**

``SELECT '1970-01-01 00:00:00' AS last_load_date;``

**Delta Load (subsequent executions):**

``SELECT MAX(transactional_date) AS last_load_date FROM core.sales;``

The logic ensures that only transactions newer than the latest loaded date are extracted and appended to the core layer.

---

## Workflow Orchestration

**Complete ETL Process**

A master workflow (complete_etl_process) orchestrates the entire data flow using KNIME’s Call Workflow (Table Based) nodes.

Execution order:
1. Staging workflows
2. Core dimension workflows
3. Core fact workflow

All workflows exchange data through Container Input/Output nodes, enabling smooth offline chaining without server-based REST execution.

---

## Results

1. End-to-end ETL pipeline successfully implemented.
2. Full and incremental loads validated.
3. Star schema with dim_product, dim_payment, and sales fact tables established.
4. AWS-hosted PostgreSQL instance used for real database operations.
5. Workflow chaining implemented through KNIME’s table-based orchestration.
6. Ready for extension into BI and reporting tools (e.g., Power BI, Tableau).

---

## Next Steps
1. Integrate Date Dimension for full time-based analytics.
2. Automate pipeline scheduling via KNIME Server or Business Hub.
3. Implement quality assurance checks for delta reconciliation.
4. Connect to a BI layer for dashboard visualisation.