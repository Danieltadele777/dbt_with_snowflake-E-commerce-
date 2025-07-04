# dbt with Snowflake E-commerce
### 📁 Project Folder Structure

```
dbt-ecommerce-project/
├── models/
│   ├── staging/
│   │   ├── stg_order.sql
│   │   ├── stg_item.sql
│   │   ├── stg_seller.sql
│   │   └── stg_product.sql
│   ├── marts/
│   │   └── ecommerce_summary.sql
│   └── marts.yml
├── seeds/
├── snapshots/
├── tests/
├── macros/
├── dbt_project.yml
├── README.md
└── .gitignore
```
Project Explanation:
- models/staging/: cleans and standardizes raw data

- models/marts/: final business-ready tables (marts)

- dbt_project.yml: defines dbt config

- tests: Contains custom SQL tests for validating business rules (e.g. no negative totals, unique keys)

- README.md: explains the project to viewers
# 🧱 dbt E-commerce Data Transformation Project

## 🔍 Overview

This project demonstrates a modular data transformation pipeline using [dbt (data build tool)](https://www.getdbt.com/) on **Snowflake**, designed to model and analyze raw e-commerce data. It applies staging, transformation, testing, and documentation best practices aligned with the dbt development workflow.

---

## 🎯 Objectives

- Clean and stage raw e-commerce datasets
- Create reusable models and apply transformations
- Build a simple **data mart** for business consumption
- Add **tests** for data quality
- Deploy in a structured **dev → prod** Snowflake environment

---

## 🧱 Project Structure

### 1. **Snowflake Setup**

I created three schemas:

| Environment      | Schema                          |
|------------------|---------------------------------|
| Raw data         | `SNOWFLAKE_LEARNING_DB.ROW_DATA`     |
| Development      | `SNOWFLAKE_LEARNING_DB.DBT_DTADELE` |
| Production       | `SNOWFLAKE_LEARNING_DB.ANALYTICS_PRODUCTION` |

---

### 2. **Source Layer (`source.yml`)**

Defined raw source tables for daily, hourly, monthly, and weekly sales:

```yaml
version: 2

sources:
  - name: genet_source
    database: SNOWFLAKE_LEARNING_DB  
    schema: DBT_DTADELE 
    tables:
      - name: order_item
        description: "Raw table loaded from CSV"
        identifier: ORDER_ITEM
        columns:
          - name: ORDER_ID
            tests:
              - not_null
              - unique
              

      - name: orders
        description: "Raw table loaded from CSV"
        identifier: ORDERS
        columns:
          - name: ORDER_ID
            tests:
              - not_null
              - unique
              
        
      - name: products
        description: "Raw table loaded from CSV"
        identifier: PRODUCTS
        columns:
          - name: PRODUCT_ID
            tests:
            - not_null
            - unique
            
      - name: sellers
        description: "Raw table loaded from CSV"
        identifier: SELLERS
        freshness: 
            warn_after: {count: 24, period: hour}
            error_after: {count: 1, period: day}
        loaded_at_field: LOADED_AT
        columns:
          - name: SELLER_ID
            tests:
            - unique
            - not_null
```

---

### 3. **Model Layer (`model.yml`)**

Defined tests as a column and model level:

```yaml      

version: 2

models:
  - name: stg_order
    description: one distinct order per row
    columns:
      - name: ORDER_ID
        description: each order_id is uniquee
        tests:
          - not_null
          - unique

  - name: stg_item
    columns:
      - name: ORDER_ID
        description: "{{ doc('order_status')}}"
        tests:
          - not_null
          - unique
```
---

### 4. **Staging Layer**
Standardized and cleaned the data:

- stg_order.sql: selected relevant order fields, formatted date

- stg_item.sql: deduplicated by order with window function

- stg_product.sql: standardized product information

- stg_seller.sql: extracted city/state, seller details

Example (stg_item.sql):

```yaml

WITH transformed AS (
  SELECT 
    order_id,
    product_id,
    seller_id,
    price,
    freight_value,
    SHIPPING_LIMIT_DATE,
    ROW_NUMBER() OVER (PARTITION BY order_id ORDER BY SHIPPING_LIMIT_DATE) AS rn
  FROM {{ source('genet_source', 'order_item') }}
)
SELECT *
FROM transformed
WHERE rn = 1

```
---

### 5. **Marts Layer**
Built a final e-commerce table using staged data.

Example (stg_item.sql):

```yaml
