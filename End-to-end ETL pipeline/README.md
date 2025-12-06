
#  AWS Serverless ETL Pipeline – Bike Store Analytics

This project implements a **fully serverless ETL data pipeline on AWS** to process, transform, and analyze bike sales data from Europe. The pipeline ingests raw CSV files, transforms them into optimized Parquet format using AWS Lambda and Pandas, catalogs them with AWS Glue, and enables analytics using Amazon Athena and visualization through Amazon QuickSight.

The architecture follows an **event-driven ETL design**, triggered automatically when new data is uploaded to Amazon S3.


## Table of Contents

- [Dataset](#dataset)
- [Architecture Overview](#architecture-overview)
- [IAM & Security Setup](#iam--security-setup)
- [S3 Data Lake Structure](#s3-data-lake-structure)
- [Event-Driven Ingestion (S3 → Lambda)](#event-driven-ingestion-s3--lambda)
- [Lambda Transformation Logic](#lambda-transformation-logic)
  - [File Validation](#1️-file-validation)
  - [Data Loading](#2️-data-loading)
  - [Data Cleaning & Transformation](#3️-data-cleaning--transformation)
  - [Parquet Conversion](#4️-parquet-conversion)
  - [Partitioned Storage](#5️-partitioned-storage)
  - [Lambda Dependencies (AWS Layer)](#lambda-dependencies-aws-layer)
  - [Lambda Role and Timeout Settings](#lambda-role-and-timeout-settings)
- [Data Cataloging with AWS Glue](#data-cataloging-with-aws-glue)
- [Query Layer – Amazon Athena](#query-layer--amazon-athena)
- [Data Quality Checks](#data-quality-checks)
  - [Null & Missing Values Check](#null--missing-values-check)
  - [Invalid Revenue & Profit Check](#invalid-revenue--profit-check)
  - [Logical Consistency Check (Revenue vs Cost)](#logical-consistency-check-revenue-vs-cost)
  - [Duplicate Records Detection](#duplicate-records-detection)
  - [Date Format Validation](#date-format-validation)
- [Business-Oriented Athena Queries](#business-oriented-athena-queries)
  - [Total Revenue by Country](#1-total-revenue-by-country)
  - [Monthly Sales Trend](#2-monthly-sales-trend)
  - [Most Profitable Product Categories](#3️-most-profitable-product-categories)
  - [Customer Demographics by Revenue](#4️-customer-demographics-by-revenue)
- [Visualization – Amazon QuickSight](#visualization--amazon-quicksight)


---

##  Dataset

**Source:** Kaggle – *Bike Sales in Europe*  
🔗 https://www.kaggle.com/datasets/sadiqshah/bike-sales-in-europe  

**Main Columns:**
Date, Day, Month, Year, Customer_Age, Age_Group, Customer_Gender,
Country, State, Product_Category, Sub_Category, Product,
Order_Quantity, Unit_Cost, Unit_Price, Profit, Cost, Revenue


**Sample Record:**
01/01/2011;1;January;2011;23;Youth (<25);M;Australia;Victoria;
Bikes;Mountain Bikes;Mountain-200 Black, 46;1;1252;2295;561;1252;1813



---
<a id="architecture-overview"></a>
##  Architecture Overview

<img width="985" height="314" alt="aws_flow" src="https://github.com/user-attachments/assets/1e13bf6d-211d-49fe-a434-fd491151a769" />



- **Pipeline Type:** ETL (Extract → Transform → Load)  
- **Processing Model:** Serverless  
- **Storage Format:** Parquet  
- **Partitioning:** Daily snapshots  
- **Query Engine:** Amazon Athena  
- **Visualization:** Amazon QuickSight  

---
<a id="iam--security-setup"></a>
##  IAM & Security Setup

- An **Admin user** creates:
  - A dedicated IAM user (`toto`)
  - Programmatic access for S3, Lambda, Glue, and Athena
- Permissions are managed through:
  - Lambda execution roles
  - Glue service role
  - Least-privilege S3 access policies

This ensures **secure and isolated access** to the data platform.

---

##  S3 Data Lake Structure

**Bucket Name:**  ```etl-bikestore```



---

##  Event-Driven Ingestion (S3 → Lambda)

When a new CSV file is uploaded to: ```s3://etl-bikestore/raw_orders/```

it automatically triggers the Lambda function:

**Lambda Name:** `manipulator`

### Trigger Configuration
- **Event Type:** `s3:ObjectCreated:*`
- **Prefix Filter:** `raw_orders/`
- **Service Principal:** `s3.amazonaws.com`

---

##  Lambda Transformation Logic

```python
def lambda_handler(event, context):

    if 'Records' not in event:
        return {
            'statusCode': 400,
            'body': json.dumps('Not an S3 event')
        }

    file_name = event['Records'][0]['s3']['object']['key']
    bucket_name = event['Records'][0]['s3']['bucket']['name']

    s3 = boto3.client('s3')

    print("Processing CSV file...")

    try:  
        response = s3.get_object(Bucket=bucket_name, Key=file_name)

        if not file_name.lower().endswith('.csv'):
            print("Not a CSV file, skipping.")
            return {'statusCode': 200, 'body': 'Skipped non-CSV file'}

        # Read file 
        csv_bytes = response['Body'].read()
        csv_string = csv_bytes.decode('utf-8')

        # Load into pandas
        df = pd.read_csv(io.StringIO(csv_string), delimiter=';')

        if df.empty:
            raise ValueError("CSV is empty")

        # Edit the dataframe
        df = manipulate_columns(df)

        # Save the dataframe to parquet
        save_to_parquet(s3, bucket_name, df)

        return {
            'statusCode': 200,
            'body': json.dumps(f'File {file_name} processed successfully')
        }

    except Exception as e:
        print("Error processing file:", e)
        return {
            'statusCode': 500,
            'body': json.dumps(f'Error processing file: {str(e)}')
        }
``` 


The Lambda function performs the following steps:

### 1️ File Validation
- Ensures the uploaded file is a `.csv`
          
```python
  if not file_name.lower().endswith('.csv'):
            print("Not a CSV file, skipping.")
            return {'statusCode': 200, 'body': 'Skipped non-CSV file'}
        }
```

### 2️ Data Loading
- Reads data from S3 using `boto3`
- Loads it into Pandas by reading first the file and then by converting into a DataFrame:
```python
  ...
        # Read file 
        csv_bytes = response['Body'].read()
        csv_string = csv_bytes.decode('utf-8')

        # Load into pandas
        df = pd.read_csv(io.StringIO(csv_string), delimiter=';')
  ...
```
### 3️ Data Cleaning & Transformation
```python
def manipulate_columns(df):
    
    # Normalize column names (strip + lower)
    df.columns = ( df.columns
               .str
               .strip()
               .str
               .lower()
               )

    # Convert date column into date format compatible with Athena
    if 'date' in df.columns:
        df['date'] = pd.to_datetime(
            df['date'],
            dayfirst=True,
            errors='coerce'
        ).dt.strftime('%Y-%m-%d')

    df['processed_at'] = pd.Timestamp.utcnow()
    
    return df
```

Normalizes column names:

  - Lowercase

  - Trim spaces

  - Converts date to Athena-compatible format (YYYY-MM-DD)

Adds metadata column:

  - ```processed_at``` (UTC timestamp)

### 4️ Parquet Conversion

```python
def save_to_parquet(s3, bucket_name, df):

    parquet_buffer = io.BytesIO()
    df.to_parquet(parquet_buffer, engine='pyarrow', compression='snappy', index=False)

    parquet_buffer.seek(0)

    now = datetime.datetime.utcnow()
    snapshot_day = now.strftime("%Y-%m-%d")
    timestamp = now.strftime("%Y%m%d-%H%M%S")

    
    key_staging = (
        f'orders_parquet_datalake/'
        f'snapshot_day={snapshot_day}/'
        f'orders_{timestamp}.parquet'
    )

    s3.put_object(Bucket=bucket_name, Key=key_staging, Body=parquet_buffer.getvalue())
    print(f'Parquet saved to s3://{bucket_name}/{key_staging}')
    
```

Converts DataFrame using:

  - pyarrow

  - snappy compression

### 5️ Partitioned Storage

Files are written with daily snapshot partitions:
```
orders_parquet_datalake/
└── snapshot_day=YYYY-MM-DD/
    └── orders_YYYYMMDD-HHMMSS.parquet
```
This enables:

  - Partition pruning in Athena
  - Faster query performance
  - Lower query costs



### Lambda Dependencies (AWS Layer)

To support Pandas, the following AWS-managed layer is used:

| Layer Name             | Python | ARN                                                                    |
| ---------------------- | ------ | ---------------------------------------------------------------------- |
| AWSSDKPandas-Python313 | 3.13   | `arn:aws:lambda:us-east-1:336392948345:layer:AWSSDKPandas-Python313:5` |

This avoids:

  - Large deployment packages

  - Dependency conflicts

  - Long cold starts

### Lambda Role and Timeout settings
It was granted ```AmazonS3FullAccess``` to the ```manipulator-role-dnllsrr``` and the **Timeout** in the Edit basic settings was increased up to 30 seconds.


## Data Cataloging with AWS Glue

A Glue crawler automatically detects schema and partitions.

**Crawler Name:** etl_bikestore_data

**IAM Role:** AWSGlueServiceRole-bikeproject

**Database:** db_bikestore

**Data Source:**
```
s3://etl-bikestore/orders_parquet_datalake/
```

**Recrawl Policy:** New folders only

**Schedule:** On-demand

After each run:

  - Schema is automatically inferred
  - Tables and partitions are updated
  - Data becomes immediately queryable in Athena


## Query Layer – Amazon Athena

Amazon Athena is used as the **serverless SQL query engine** for this project. It allows running standard ANSI SQL queries directly on top of data stored in Amazon S3 without requiring any infrastructure management.

In this pipeline, Athena queries the **Parquet-based analytical data lake** stored in:

```
s3://etl-bikestore/query_results/
```
and uses the **AWS Glue Data Catalog** as the metastore to resolve table schemas and partitions.


## Data Quality Checks
To ensure the reliability and correctness of analytical results, multiple **data quality validation checks** are performed directly in Amazon Athena. These checks help detect ingestion errors, transformation issues, and anomalous business values.


###  Null & Missing Values Check

**Business Risk:**  
Null values in key fields can break dashboards and lead to incorrect aggregations.

```sql
SELECT 
    COUNT(*) AS total_rows,
    SUM(CASE WHEN date IS NULL THEN 1 ELSE 0 END) AS null_dates,
    SUM(CASE WHEN revenue IS NULL THEN 1 ELSE 0 END) AS null_revenue,
    SUM(CASE WHEN country IS NULL THEN 1 ELSE 0 END) AS null_country,
    SUM(CASE WHEN product_category IS NULL THEN 1 ELSE 0 END) AS null_product_category
FROM db_bikestore.bike_orders;
```

### Invalid Revenue & Profit Check

Business Risk:
Negative revenue or profit usually indicates corrupted data or ingestion issues.
```sql
SELECT *
FROM db_bikestore.bike_orders
WHERE revenue < 0
   OR profit < 0;
```

### Logical Consistency Check (Revenue vs Cost)

**Business Rule:**
Revenue should always be greater than or equal to cost.
```sql
SELECT *
FROM db_bikestore.bike_orders
WHERE revenue < cost;
```


### Duplicate Records Detection

**Business Risk:**
Duplicate rows inflate revenue and distort all KPIs.
```sql
SELECT 
    date,
    product,
    country,
    COUNT(*) AS duplicate_count
FROM db_bikestore.bike_orders
GROUP BY date, product, country
HAVING COUNT(*) > 1;
```


### Date Format Validation

**Business Risk:**
Malformed dates break time-series analysis and partition filtering.
```sql
SELECT *
FROM db_bikestore.bike_orders
WHERE TRY_CAST(date AS DATE) IS NULL;
```


##  Business-Oriented Athena Queries

Below are real-world business queries that can be executed on the `bike_orders` Athena table.

---

### 1) Total Revenue by Country

Which countries generate the highest revenue?

```sql
SELECT 
    country,
    SUM(revenue) AS total_revenue
FROM db_bikestore.bike_orders
GROUP BY country
ORDER BY total_revenue DESC;
```


### 2) Monthly Sales Trend

How does revenue evolve over time?
```sql
SELECT 
    year,
    month,
    SUM(revenue) AS monthly_revenue
FROM db_bikestore.bike_orders
GROUP BY year, month
ORDER BY year, month;
```

### 3️) Most Profitable Product Categories
Which product categories generate the most profit?

```sql
SELECT 
    product_category,
    SUM(profit) AS total_profit
FROM db_bikestore.bike_orders
GROUP BY product_category
ORDER BY total_profit DESC;
```

### 4️) Customer Demographics by Revenue
Which customer age groups generate the most revenue?

```sql
SELECT 
    age_group,
    SUM(revenue) AS total_revenue
FROM db_bikestore.bike_orders
GROUP BY age_group
ORDER BY total_revenue DESC;
```




## Visualization – Amazon QuickSight

QuickSight connects to Athena to create dashboards such as:

  - Revenue by time

  - Sales by country and category

  -  Profit distribution

  - Customer demographics

This completes the end-to-end analytics workflow from raw ingestion to BI dashboards.
