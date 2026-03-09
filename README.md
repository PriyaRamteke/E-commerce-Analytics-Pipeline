# 🛍️ E-commerce Analytics Pipeline

## 📊 Project Overview

End-to-end **data engineering pipeline** built on **Databricks** that processes e-commerce transactional data through a medallion architecture (Bronze → Silver → Gold), integrates with **Google BigQuery**, and visualizes insights using **Looker Studio**.

This project demonstrates production-grade data engineering practices including:
* **Medallion Architecture** (Bronze/Silver/Gold layers)
* **Data Quality Checks & Schema Validation**
* **Automated ETL Orchestration** using Databricks Jobs
* **Cloud Data Warehouse Integration** (BigQuery)
* **Business Intelligence Dashboards** (Looker Studio)
* **Advanced Analytics** (RFM Segmentation, Fraud Detection, Sentiment Analysis)

---

## 🏗️ Architecture

```
┌─────────────────┐
│  Raw CSV Files  │ (transactions, customers, products, stores, promotions, feedbacks)
└────────┬────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────────┐
│                    DATABRICKS LAKEHOUSE                         │
│                                                                 │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐    │
│  │ 🟤 BRONZE    │───▶│ ⚪ SILVER    │───▶│ 🟡 GOLD      │    │
│  │ Raw Data     │    │ Cleaned      │    │ Aggregated   │    │
│  │ Validated    │    │ Enriched     │    │ Business KPIs│    │
│  └──────────────┘    └──────────────┘    └──────────────┘    │
│                                                                 │
│  [Orchestrated by Databricks Workflow with 3 Tasks]            │
└───────────────────────────┬─────────────────────────────────────┘
                            │
                            ▼
                ┌───────────────────────┐
                │  Google BigQuery      │ (Cloud Data Warehouse)
                └───────────┬───────────┘
                            │
                            ▼
                ┌───────────────────────┐
                │   Looker Studio       │ (Interactive Dashboards)
                └───────────────────────┘
```

---

## 🛠️ Tech Stack

| Component | Technology |
|-----------|------------|
| **Platform** | Databricks (Serverless Compute) |
| **Processing** | Apache Spark (PySpark) |
| **Storage** | Databricks Volumes (Unity Catalog) |
| **Format** | Delta Lake |
| **Cloud Provider** | Google Cloud Platform (GCP) |
| **Data Warehouse** | Google BigQuery |
| **BI Tool** | Looker Studio |
| **Orchestration** | Databricks Jobs & Workflows |
| **Language** | Python 3.x, SQL |

---

## 📁 Project Structure

```
ecommerce-analytics-pipeline/
│
├── README.md                          # Project documentation
├── screenshots/                       # Pipeline & Dashboard screenshots
│   ├── pipeline-run.png              # Databricks job execution
│   └── looker-dashboard.png          # Looker Studio visualizations
│
├── notebooks/
│   ├── 01-bronze-ingestion.ipynb     # Data ingestion & schema validation
│   ├── 02-silver-cleaning.ipynb      # Data cleaning & enrichment
│   ├── 03-gold-analytics.ipynb       # Business KPIs & aggregations
│   └── 04-bigquery-integration.ipynb # BigQuery export & integration
│
├── data/                              # Sample raw CSV files
│   ├── transactions.csv
│   ├── customers.csv
│   ├── products.csv
│   ├── stores.csv
│   ├── promotions.csv
│   └── feedbacks.csv
│
└── config/
    └── gcp-service-account.json      # GCP credentials (DO NOT COMMIT)
```

---

## 📚 Pipeline Stages

### 🟤 Stage 1: Bronze Layer - Data Ingestion (`Notebook 01`)

**Purpose:** Raw data ingestion with explicit schema enforcement

**Key Operations:**
* Read 6 CSV files (transactions, customers, products, stores, promotions, feedbacks)
* Apply explicit PySpark schemas (`StructType`) for data validation
* Perform initial data quality checks (null validation, duplicate detection)
* Write to Delta Lake format in Bronze layer

**Output:** 6 validated Delta tables in `/Volumes/workspace/default/eccomerce/bronze/`

**Code Highlights:**
```python
# Schema validation example
txn_schema = StructType([
    StructField('transaction_id', IntegerType(), True),
    StructField('customer_id', IntegerType(), True),
    StructField('total_amount', DoubleType(), True),
    ...
])

txn_df = spark.read.option('header', True).schema(txn_schema).csv(RAW_PATH+'transactions.csv')
txn_df.write.mode('overwrite').format('delta').save(BRONZE_PATH+'transactions')
```

---

### ⚪ Stage 2: Silver Layer - Cleaning & Normalization (`Notebook 02`)

**Purpose:** Data cleansing, deduplication, and enrichment through joins

**Key Operations:**
* Remove duplicates based on `transaction_id`
* Standardize timestamp formats (`dd-MM-yyyy`)
* Handle missing values with `coalesce()` and default values
* Enrich transactions by joining with dimension tables:
  * Customer demographics
  * Product details
  * Store locations
  * Promotion discounts
  * Customer feedback/ratings
* Calculate derived columns:
  * `final_amount` (after discount)
  * `promo_in_range` (promotion validity check)
  * `_ingest_time` (audit timestamp)
* **Data Quality Checks:** Fail pipeline if >10% invalid stores

**Output:** Partitioned Delta table `transactions_enriched` in `/Volumes/workspace/default/eccomerce/silver/`

**Code Highlights:**
```python
# Enrichment with multiple joins
silver = (txn_clean.alias('t')
    .join(cust.alias('c'), col('t.customer_id') == col('c.cust_id'), 'left')
    .join(prod.alias('p'), 'product_id', 'left')
    .join(store.alias('s'), 'store_id', 'left')
    .join(promo.alias('pr'), 'product_id', 'left')
    .join(fb.alias('f'), ..., 'left')
)

# Data quality validation
if (invalid_store_count / total) > 0.10:
    raise Exception("DQ check failed: > 10% invalid store")
```

---

### 🟡 Stage 3: Gold Layer - Business KPIs & Analytics (`Notebook 03`)

**Purpose:** Generate business-ready aggregations and advanced analytics

**Gold Tables Created:**

#### 1. **Daily Store Category Sales**
* **Metrics:** `gross_sales`, `net_sales`, `unique_customers`, `txn_count`
* **Dimensions:** `transaction_date`, `store_name`, `category`
* **Use Case:** Track daily revenue by store and product category

#### 2. **Top Customers**
* **Metrics:** `total_spent`, `txn_count`
* **Dimensions:** `customer_id`, `customer_name`, `store_name`, `region`
* **Use Case:** Identify high-value customers for retention campaigns

#### 3. **Promotion Impact Analysis**
* **Metrics:** `promo_txns`, `promo_sales`, `avg_with_promo`
* **Dimension:** `promotion_id`
* **Use Case:** Measure ROI of promotional campaigns

#### 4. **Product Sentiment Analysis**
* **Metrics:** `avg_rating`, `rating_count`
* **Dimensions:** `product_id`, `product_name`, `category`
* **Use Case:** Identify products needing improvement

#### 5. **RFM Customer Segmentation**
* **Metrics:** `recency_days`, `frequency`, `monetary`
* **Buckets:** Categorize customers into High/Medium/Low segments
* **Use Case:** Personalized marketing and customer lifecycle management

#### 6. **Fraud Detection (Suspect Customers)**
* **Logic:** Flag transactions with:
  * Time difference < 30 minutes between transactions
  * Different stores
  * High transaction amounts (> $1000)
* **Use Case:** Real-time fraud alerting system

**Output:** 6 Delta tables in `/Volumes/workspace/default/eccomerce/gold/`

---

### 🔗 Stage 4: BigQuery Integration (`Notebook 04`)

**Purpose:** Export analytics to Google BigQuery for cloud-based querying and BI integration

**Key Operations:**
* Authenticate with GCP service account (JSON key)
* Read Gold layer Delta tables
* Export to CSV (temporary format)
* Load into BigQuery dataset: `decisive-cinema-489218-g0.ecommerce`

**BigQuery Tables Created:**
* `daily_store_category`
* `top_customers`
* `promo_impact`
* `product_sentiment`
* `rfm`

**Looker Studio Integration:**
1. Connect Looker Studio to BigQuery
2. Select project → dataset → tables
3. Create interactive dashboards with drag-and-drop
4. Share reports with stakeholders

**Code Highlights:**
```python
CREDENTIALS_FILE = "/Volumes/workspace/default/eccomerce/<service-account>.json"

# Read Gold layer data
for table_name, delta_path in tables.items():
    df = spark.read.format('delta').load(delta_path)
    df.write.option('credentialsFile', CREDENTIALS_FILE) \
        .format('bigquery') \
        .option('table', f'{GCP_PROJECT}.{BQ_QUERY}.{table_name}') \
        .save()
```

---

## 🔄 Automated Workflow

**Databricks Job:** `Ecommerce Analytics Pipeline`

**Tasks:**
1. **Data_Ingestion** → Executes Notebook 01 (Bronze)
2. **Cleaning_and_Normalization** → Executes Notebook 02 (Silver)
3. **Analysis** → Executes Notebook 03 (Gold)

**Execution:** 
* **Trigger:** Manual or scheduled (daily/hourly)
* **Compute:** Serverless (cluster-terminated)
* **Duration:** ~25 seconds (from screenshot)
* **Status:** ✅ All tasks succeeded

*(See `screenshots/pipeline-run.png` for execution details)*

---

## 📸 Screenshots

### 1. Databricks Pipeline Execution
![Pipeline Run](screenshots/pipeline-run.png)

**Shows:**
* 3-stage workflow (Data_Ingestion → Cleaning_and_Normalization → Analysis)
* Task execution times (49s, 16s, 18s)
* Job ID: `1058820002275406`
* Serverless compute configuration

### 2. Looker Studio Dashboard
![Looker Dashboard](screenshots/looker-dashboard.png)

**Shows:**
* Interactive pie charts for promotion analysis
* Real-time data from BigQuery
* Dimensions: `promo_impact`, `avg_with_promo`, `promo_sales`

---

## 🚀 Setup Instructions

### Prerequisites
* Databricks workspace (GCP)
* Google Cloud Platform account
* BigQuery API enabled
* Looker Studio access

### Step 1: Clone Repository
```bash
git clone https://github.com/<your-username>/ecommerce-analytics-pipeline.git
cd ecommerce-analytics-pipeline
```

### Step 2: Upload Data to Databricks
1. Navigate to Databricks Workspace → **Volumes**
2. Create catalog: `workspace`, schema: `default`, volume: `eccomerce`
3. Upload CSV files from `data/` folder to `/Volumes/workspace/default/eccomerce/`

### Step 3: Import Notebooks
1. Go to **Workspace** → Import
2. Upload all 4 notebooks from `notebooks/` folder
3. Update paths in Cell 1 if needed:
   ```python
   RAW_PATH = '/Volumes/workspace/default/eccomerce/'
   BRONZE_PATH = '/Volumes/workspace/default/eccomerce/bronze/'
   SILVER_PATH = '/Volumes/workspace/default/eccomerce/silver/'
   GOLD_PATH = '/Volumes/workspace/default/eccomerce/gold/'
   ```

### Step 4: Configure GCP Authentication
1. Create a GCP service account with BigQuery permissions
2. Download JSON key file
3. Upload to `/Volumes/workspace/default/eccomerce/`
4. Update `CREDENTIALS_FILE` path in Notebook 04
5. Update `GCP_PROJECT` and `TEMP_GCS_BUCKET` variables

### Step 5: Create Databricks Workflow
1. Go to **Workflows** → Create Job
2. Add 3 tasks:
   * Task 1: Run Notebook 01
   * Task 2: Run Notebook 02 (depends on Task 1)
   * Task 3: Run Notebook 03 (depends on Task 2)
3. Configure task dependencies (linear execution)
4. Set compute: Serverless or existing cluster

### Step 6: BigQuery Setup
1. Go to BigQuery console
2. Create dataset: `ecommerce`
3. Run Notebook 04 to export tables
4. Verify tables in BigQuery UI

### Step 7: Looker Studio Dashboard
1. Open [Looker Studio](https://lookerstudio.google.com/)
2. Create New Report → Add Data Source
3. Select **BigQuery Connector**
4. Authorize and select project → `ecommerce` dataset
5. Add tables and create visualizations

---

## 📊 Key Business Insights

* **Revenue Optimization:** Track daily sales by store and category to identify top performers
* **Customer Retention:** Use RFM segmentation to target at-risk customers
* **Promotion Effectiveness:** Measure discount impact on sales velocity
* **Product Quality:** Monitor sentiment scores to prioritize product improvements
* **Fraud Prevention:** Detect suspicious transaction patterns in real-time

---

## 🎯 Data Engineering Best Practices Demonstrated

✅ **Medallion Architecture** - Bronze/Silver/Gold layering for data quality progression  
✅ **Schema Enforcement** - Explicit schema validation prevents bad data ingestion  
✅ **Data Quality Checks** - Automated validation gates in transformation pipeline  
✅ **Partitioning** - Optimized query performance with date partitioning  
✅ **Idempotency** - Overwrite mode ensures repeatable pipeline runs  
✅ **Incremental Processing** - Ready for CDC (Change Data Capture) implementation  
✅ **Observability** - Audit columns (`_ingest_time`) for lineage tracking  
✅ **Error Handling** - Exception-based DQ failures prevent bad data propagation  

---

## 🔮 Future Enhancements

* **Real-time Streaming:** Replace batch processing with Spark Structured Streaming
* **Machine Learning:** Customer churn prediction using MLflow
* **Data Catalog:** Implement Unity Catalog for governance and lineage
* **Alerting:** Integrate with PagerDuty/Slack for pipeline failures
* **dbt Integration:** Version-controlled SQL transformations
* **CI/CD:** GitHub Actions for automated notebook deployment

---

## 🙏 Acknowledgments

* Databricks Community Edition for serverless compute
* Google Cloud Platform for BigQuery integration
* Apache Spark open-source community

---

