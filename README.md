# 🛒 UK Online Retail — Data Pipeline with Medallion Architecture

Data engineering project built on Databricks using PySpark and Amazon S3, following the Medallion Architecture (Raw → Bronze → Silver → Gold). The project also includes the design of a dimensional model to support analytical workloads.

---

## 🗂️ Dataset

The dataset used is **OnlineRetail.csv**, a public dataset of real e-commerce transactions from the United Kingdom, containing information such as invoice number, product code and description, quantity, date, unit price, customer, and country.

---

## 💼 Business Understanding

1. What is the main business process represented by the dataset?  
The primary business process represented in the dataset is the sale of products to customers, recorded through purchase transactions.

2. What does a sale represent in this context?  
A sale represents a commercial transaction in which a customer purchases one or more products. In the dataset, each sale is identified by an InvoiceNo and may contain multiple invoice line items. Records whose InvoiceNo begins with the letter "C" indicate cancellations and, therefore, do not represent completed sales.

3. What should the granularity of the fact table be?  
The granularity of the fact table is at the invoice line item level. Each record represents a specific product sold within a transaction (InvoiceNo), including attributes such as quantity sold and unit price.
---

## 📐 Fact Table Granularity

The choice to have each record in the fact table represent one invoice line item is based on the fact that this is the lowest level of granularity available in the dataset. This approach preserves all transaction details, enabling future aggregations at different levels, such as by customer, product, country, or transaction, without losing information.

---

## 🏗️ Architecture

```mermaid
flowchart TD
    A["Amazon S3<br/>OnlineRetail.csv"] --> B["Raw Layer<br/>S3 Landing Volume"]

    B --> C["Bronze Layer<br/>Delta Table<br/>uk_ecommerce.bronze.online_retail"]

    C --> D["Silver Layer<br/>Delta Table<br/>Cleaned and enriched data<br/>uk_ecommerce.silver.online_retail"]

    I["normalized_descriptions.csv<br/>Reference Volume"] -.-> D

    D --> E["Gold Layer<br/>Aggregated analytical tables"]

    E --> F["sales_by_customer"]
    E --> G["sales_per_product"]
    E --> H["sales_per_country"]

    B -. "ingestion-raw-bronze" .-> C
    C -. "ingestion-bronze-silver" .-> D
    D -. "ingestion-silver-gold" .-> E
    I -. "description normalization" .-> D
```

---

## 📊 Data Model

The dimensional model is designed to support analytical queries by organizing descriptive information into dimensions. Each dimension provides business context for the fact table while reducing data redundancy and improving query performance.

---

### Customer Dimension

<p align="center">
  <img src="assets/diagrams/pngs/dim_customer.png" width="300">
</p>

The `dim_customer` table stores descriptive information about customers and provides customer context for analytical queries.

#### Attributes

| Column         | Description                                  |
| -------------- | -------------------------------------------- |
| `customer_key` | Surrogate key generated for the dimension.   |
| `customer_id`  | Customer identifier from the source dataset. |
| `country`      | Customer's country.                          |

#### Modeling Decisions

The original dataset contains only two customer-related attributes: `CustomerID` and `Country`. Since `Country` provides valuable geographical context, it was incorporated into the customer dimension to enable analyses such as sales by country and customer distribution without requiring additional transformations.

---

### Product Dimension

<p align="center">
  <img src="assets/diagrams/pngs/dim_product.png" width="300">
</p>

The `dim_product` table stores descriptive information about the products available in the dataset.

#### Attributes

| Column        | Description                                 |
| ------------- | ------------------------------------------- |
| `product_key` | Surrogate key generated for the dimension.  |
| `stock_code`  | Product identifier from the source dataset. |
| `description` | Product description.                        |

---

### Date Dimension

<p align="center">
  <img src="assets/diagrams/pngs/dim_date.png" width="320">
</p>

The `dim_date` table stores calendar attributes used to support time-based analysis and reporting.

#### Attributes

| Column       | Description                                |
| ------------ | ------------------------------------------ |
| `date_key`   | Surrogate key generated for the dimension. |
| `date`       | Calendar date.                             |
| `day`        | Day of the month.                          |
| `month`      | Month number.                              |
| `month_name` | Month name.                                |
| `year`       | Calendar year.                             |

#### Why Pre-compute a Date Dimension?

Unlike transactional data, calendar dates exist independently of business events. Therefore, the date dimension can be generated in advance rather than being created dynamically during data ingestion.

Pre-computing the date dimension simplifies analytical queries, ensures consistency across reports, and allows complete time-series analysis, including dates without recorded sales.

---

### Fact Table
<p align="center">
  <img src="assets/diagrams/pngs/fact_table.png" width="320">
</p>

| Column       | Description                                      |
|--------------|--------------------------------------------------|
|`sale_key`    |Surrogate Key (PK)                                |
|`customer_key`|Foreign Key (FK) to `dim_customer`                |
|`product_key` |Foreign Key (FK) to `dim_product`                 |
|`date_key`    |Foreign Key (FK) to `dim_date`                    |
|`quantity`    |Quantities of each product (item) per transaction |
|`unit_price`  |Unit price of the product                         |
|`total_sales` |Total sales amount (`quantity * unit_price`)      |
|`invoice_no`  |6-digit identifier assigned to each transaction   |

---

### Star Schema
<p align="center">
  <img src="assets/diagrams/pngs/star_schema.png" width="520">
</p>

The star schema connects the `fact_sales` table with the dimensional tables used to describe each sales transaction.

---

## Data Quality Analysis

After removing null and duplicate records from the dataset, an exploratory analysis identified **213 `StockCode` values** associated with multiple product descriptions.

The inconsistencies include:

* Minor differences in formatting or wording.
* Possible historical changes in product descriptions.
* Descriptions that differ significantly, making it impossible to determine whether they refer to the same product or different products.

Since the dataset does not provide a master product reference, standardization could not be inferred automatically. Each of the 213 divergent cases was reviewed manually to define a single canonical description per `StockCode`, and the result was stored as a reference file (`normalized_descriptions.csv`).

During the Silver → Gold step, this reference is joined back into `df_silver` and used to overwrite the `Description` column via `coalesce`, so that products with a validated normalization receive the canonical description while all other records keep their original value unchanged.

---

## Country Field Analysis

An exploratory analysis identified the **United Kingdom** as the country with the highest number of distinct customers (3,950), consistent with this being a UK-based online retailer and its primary market.

The `Country` field was also checked for formatting inconsistencies (casing variations and typos). After normalizing case and trimming whitespace, every distinct country name mapped to exactly one original spelling — confirming the field is already well standardized, with no corrections needed.

---

## Cancellation Timing Analysis

For each cancellation, the closest matching original sale (same `CustomerID` and `StockCode`, occurring earlier) was identified, resulting in 7,069 matched cancellation-sale pairs.

- **Average time to cancellation:** 29.27 days
- **Median time to cancellation:** 12 days
- **Range:** from 1 day (minimum) to 368 days (maximum) — the 368-day case falls within the dataset's own time span (2010-12-01 to 2011-12-09), confirming it reflects a genuine long-standing purchase rather than a matching error

The gap between the average and the median indicates a right-skewed distribution: most cancellations happen soon after purchase, while a smaller number of long-delayed cases pull the average upward. A histogram of the full distribution confirms this, with a sharp concentration in the first 0-20 days.

Using the IQR method (Q1 = 6 days, Q3 = 29 days, IQR = 23), cancellations occurring more than **63.5 days** after the original sale were classified as statistical outliers. These represent **834 out of 7,069** matched cancellations (**11.80%**). A closer look at this outlier group shows it is not random noise: it still decays gradually from 60 days onward, with a small resurgence near 350 days that overlaps with the dataset's maximum observed delay.

---

## 🔄 Pipeline Steps

### 1. Raw → Bronze (`ingestion-raw-bronze`)

Reads the CSV stored in the S3 volume (`/Volumes/uk_ecommerce/raw/landing/OnlineRetail.csv`) and persists it as a Delta Table in the Bronze schema.

- CSV reading with schema inference (`inferSchema=True`)
- Cast of the `CustomerID` column to `string`
- Write to `uk_ecommerce.bronze.online_retail`

### 2. Bronze → Silver (`ingestion-bronze-silver`)

Cleaning and transformation of the raw data.

- Removal of duplicate records (`dropDuplicates`)
- Removal of null values (`dropna`)
- Creation of the calculated column `totalSales` (`Quantity * UnitPrice`)
- Parsing and normalization of the `InvoiceDate` column to `DateType`
- Write to `uk_ecommerce.silver.online_retail`

### 3. Silver → Gold (`ingestion-silver-gold`)

Product description normalization and analysis of order cancellations.

- Join of `df_silver` with a manually validated reference file (`normalized_descriptions.csv`), used to standardize the `Description` column for `StockCode` values with multiple descriptions
- Matching of each cancellation (`InvoiceNo` starting with "C") to its closest original sale, based on matching `CustomerID` and `StockCode`
- Calculation of the elapsed time (in days) between an original sale and its cancellation, including average, median, minimum, and maximum
- Statistical outlier detection on cancellation delay using the IQR method

---

## 🛠️ Stack

| Tool | Usage |
|---|---|
| **Databricks** | Notebook execution environment |
| **PySpark** | Distributed data processing |
| **Delta Lake** | Table storage format |
| **Amazon S3** | Source file storage (landing zone) |

---

## 📂 Project Structure

```text
uk-online-retail/
├── assets/
│   ├── architecture/
│   └── diagrams/
│       ├── pngs/
│       └── drawio/
├── notebooks/
│   ├── ingestion-raw-bronze.ipynb
│   ├── ingestion-bronze-silver.ipynb
│   └── ingestion-silver-gold.ipynb
├── warehouse/
│   ├── diagrams/
│   └── dimensions/
└── README.md
```

---

## 🚀 Implementation

### Amazon S3 Landing Zone

<p align="center">
  <img src="assets/architecture/s3-bucket.png" width="900">
</p>

The original `OnlineRetail.csv` dataset is stored in an Amazon S3 bucket, which serves as the landing zone for the ingestion pipeline. From this location, the data is accessed by Databricks and loaded into the Raw layer before progressing through the Medallion Architecture.

### Unity Catalog Structure

<p align="center">
  <img src="assets/architecture/unity-catalog.png" width="450">
</p>

The project is organized in the Databricks Unity Catalog following the Medallion Architecture. The catalog contains the Raw, Bronze, Silver, and Gold schemas, where each layer represents a different stage of data processing—from raw ingestion to business-ready analytical tables.
