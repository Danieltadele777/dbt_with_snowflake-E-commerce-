# 🧱 dbt E-commerce Data Transformation Project
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
## Table of contents
 - [Project Explanation](#project-explanation)
 - [Overview](#overview)
 - [Objectives](#objectives)
 - [Project Structure](#project-structure)
 - [Documentation](#documentation)
 - [Final Outcome](#final-outcome)
 - [Skills Demonstrated](#skills-demonstrated)


## [Project Explanation]()

- models/staging/: cleans and standardizes raw data

- models/marts/: final business-ready tables (marts)

- dbt_project.yml: defines dbt config

- tests: Contains custom SQL tests for validating business rules (e.g. no negative totals, unique keys)

- README.md: explains the project to viewers



##  [Overview]()

This project demonstrates a modular data transformation pipeline using [dbt (data build tool)](https://www.getdbt.com/) on **Snowflake**, designed to model and analyze raw e-commerce data. It applies staging, transformation, testing, and documentation best practices aligned with the dbt development workflow.

---

## [Objectives]()

- Clean and stage raw e-commerce datasets
- Create reusable models and apply transformations
- Build a simple **data mart** for business consumption
- Add **tests** for data quality
- Deploy in a structured **dev → prod** Snowflake environment

---

## [Project Structure]()

### 1. **Snowflake Setup**

I created three schemas:

| Environment      | Schema                          |
|------------------|---------------------------------|
| Raw data         | `SNOWFLAKE_LEARNING_DB.ROW_DATA`     |
| Development      | `SNOWFLAKE_LEARNING_DB.DBT_DTADELE` |
| Production       | `SNOWFLAKE_LEARNING_DB.ANALYTICS_PRODUCTION` |

---

### 2. **Source Layer (`source.yml`)**

Defined raw source tables for item, order, product, and seller e-commerce datasets:

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

Example (dim_e_commerce.sql):

```yaml
{{
    config(materialized='table')
}}

WITH ITEM AS (
  SELECT *
  FROM {{ 
           ref("stg_item")
  }}
),

SELLER AS (
  SELECT *
  FROM 
     {{
        ref("stg_seller")
     }}
),

ORDER_TABLE AS (
  SELECT *
  FROM 
     {{
        ref("stg_order")
     }}
),

PRODUCT AS (
  SELECT *
  FROM 
    {{
         ref("stg_product")
    }}
),

TOTAL AS (
  SELECT
    i.order_id,
    i.product_id,
    i.seller_id,
    i.price,
    i.freight_value,
    s.seller_city,
    s.seller_state,
    o.customer_id,
    o.order_status,
    o.estimated_date,
    p.product_category_name
  FROM ITEM i
  JOIN SELLER s ON i.seller_id = s.seller_id
  JOIN ORDER_TABLE o ON i.order_id = o.order_id
  JOIN PRODUCT p ON p.product_id = i.product_id
)

SELECT *
FROM TOTAL

```
---

### 6. **Data Quality tests at Source and Model levels**
Created data quality tests at the source and model levels. The model-level data quality checks are used to check data tests like uniqueness and non-null values of my models. The source-level data tests I used it to check our row data ingested from an outside source, if it fits our data test logic.

Here is an example for a data test at the model level:

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
        description: "{{ doc('order_status')}}" ### for learning purpose only
        tests:
          - not_null
          - unique
```
---
I also created custom-based SQL logic test to check the price column:

```yaml
with item as (
    select *
    from {{ ref('stg_item') }}
)

select 
  order_id,
  sum(price) as total_price
from item
group by order_id
having total_price < 0

```
---
### 6. [Documentation]()

I created multiple documentation at the model level, column level, and source level. Here is an example of my documentation at source level:

```yaml

{% docs order_status %}

| status         | definition                                                    |
|----------------|---------------------------------------------------------------|
| placed         | Order placed but not yet shipped                              |
| shipped        | Order has been shipped but hasn't yet been delivered          |
| completed      | Order has been received by customers                          |
| return_pending | Customer has indicated they would like to return this item    |
| returned       | Item has been returned                                        |

{% enddocs %}

```
---

### 7. [Final Outcome]()
I used dbt run and dbt build to compile and execute the models in the targeted Snowflake warehouse. I also configured a production environment in dbt and scheduled jobs to run at specific hours. 

After execution, I successfully verified that the models were materialized in the ANALYTICS_PRODUCTION schema in Snowflake.

### 8. [Skills Demonstrated]()

- dbt Cloud & CLI

- SQL transformation logic

- Snowflake warehouse integration

- Jinja templating & YAML config

- Testing & documentation in dbt

- Data modeling best practices (modular layers)
