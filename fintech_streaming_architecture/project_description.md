# Streaming Data Pipeline for a FinTech Company
## Business Problem
ABC FinTech’s hourly batch fraud checks allow fraudulent transactions to slip through. Goal: sub‑second fraud detection, event‑driven alerts, lineage for compliance, and elastic scaling.
## Current Ecosystem
 - **Storage**: on-prem Oracle
 - **ETL**: cron‑driven Python
 - **Fraud Rules**: hourly batch

## Additional Technical & Business Requirements

1. **Current sources(API Based):** POS terminals, e‑commerce, SAP ERP, Salesforce CRM, mobile, weather/pricing APIs
2. **Daily volume:** 5 – 10 M rows / ≈ 2 TB per day
3.  **Data formats:** JSON, CSV/Parquet, AVRO/ORC
4.  **Real‑time feeds:** POS, e‑comm, inventory sensors (treat transactions as “POS”)
5.  **Compliance:** PII present → encryption/masking/RBAC/audits
6.  **Platform preference:** AWS preferred; Azure/GCP possible
7.  **Analytics tools:** Power BI/Tableau/Looker; SageMaker/Databricks for ML fraud models
8.  **Growth (1‑3 yrs):** 3‑5× growth in 3 yrs
9.   **Existing use‑cases:** abandoned cart, store dash, inventory opt., customer segmentation (plus now fraud detection)
10. **Time‑travel:** required
11. **Primary users:** analysts, fraud ops, data scientists
12. **Success metrics:** latency < 1 s, self‑service reports, fraud‑alert precision/recall, CSAT
13.  **API layer:**  REST, GraphQL optional, Kafka REST Proxy

## Proposed Architecture
This modern, event-driven data pipeline leverages key **AWS** services to move from hourly batch processing to sub-second, real-time fraud detection. High-volume, low-latency data streams are ingested through **Amazon API Gateway** and **Lambda** to a resilient message broker, **Amazon MSK**. From there, **Kinesis Data Analytics for Apache Flink** serves as the core stream processing engine, performing real-time analysis and publishing fraud alerts to a dedicated **Kafka** topic. A tiered storage approach uses **Amazon DynamoDB** for sub-second lookups, **Amazon S3** as the durable data lake for long-term storage and immutability, and **Snowflake** as a data warehouse for business intelligence. This design supports both immediate fraud checks and historical analysis, with **Snowflake** providing time-travel capabilities for audit and compliance.
<img width="1200" height="1172" alt="fintech diagram drawio" src="https://github.com/user-attachments/assets/c58b81fd-04e6-468e-a600-9542e6e2a14c" />

### Data Ingestion & Collection
This layer is responsible for ingesting diverse data formats from various sources with high throughput.
- **Message Broker: Amazon MSK** (Managed Streaming for Apache Kafka). Apache Kafka is the ideal choice due to its robustness, high throughput, and native support for multiple data formats (JSON, CSV, Avro) and protocols (Kafka REST Proxy). It will collect real-time data from POS terminals, e-commerce platforms, and IoT sensors.
- **APIs & Webhooks:** For sources like Salesforce, SAP ERP, and weather APIs, Amazon API Gateway will be used to create secure, scalable REST or GraphQL endpoints. A Lambda function will act as a proxy to ingest data, including payloads from webhooks, directly into the Kafka topics.
- **Historical Data:** Existing on-prem Oracle and Python ETL jobs will be migrated to a batch ingestion process using AWS Glue jobs to periodically dump data into S3 for historical analysis.

### Stream Processing & Analysis
This is the core of the real-time fraud detection engine, processing events in-flight with sub-second latency.
- **Stream Processing Engine: Amazon Kinesis Data Analytics for Apache Flink**. Flink is chosen for its native support for complex event processing (CEP), stateful stream processing, and ability to handle "time-travel" (event time vs. processing time) for historical context. It will:
  - Perform real-time fraud rule checks on streaming data.
  - Join transaction streams with customer segmentation and inventory data.
  - Publish fraud alerts to a dedicated Kafka topic.
- **Machine Learning Integration: Amazon SageMaker** will be used to train and deploy real-time ML fraud models. The Flink application can invoke these deployed models via an API endpoint for advanced, predictive fraud scoring.
### Data Storage & Management
A tiered storage strategy ensures data is available for both real-time fraud checks and long-term analysis.
- **Real-time Storage: Amazon DynamoDB** will store recent transaction data and user sessions. Its low-latency read/write performance is essential for sub-second fraud checks and for the Flink application to quickly access state.
- **Data Lake: Amazon S3** (Simple Storage Service) will be the central data lake, storing all raw and processed data in open formats like Parquet and Avro. This provides cost-effective, durable storage and serves as the single source of truth.
- **Data Warehouse: Snowflake** will house transformed, clean, and modeled data for BI tools and analysts. It's optimized for complex SQL queries and reporting.
### Data Consumption & Business Intelligence
This layer provides self-service access and reporting for various users.
- **BI & Reporting: Snowflake** will serve as the data source for BI tools like Power BI.
- **Fraud Operations:** A dashboard application will subscribe to the fraud alert Kafka topic, displaying real-time alerts
- **Data Scientists:** They will access the raw and curated data in the **S3** data lake via **Amazon SageMaker** on AWS for developing new ML models and performing ad-hoc analysis.
### Security, Governance, & Scalability
- **Security: AWS IAM** for role-based access control (RBAC), **AWS KMS** for encryption of data at rest and in transit, and **AWS Glue Data Catalog** for metadata management and PII discovery. Automated audits and logging will be handled by **AWS CloudTrail** and **Amazon GuardDuty**.
- **Scalability:** All chosen services (MSK, Flink, S3) are managed and scale elastically and automatically to accommodate the 3-5x growth projection.
- **Time-Travel:** The use of **Snowflake** allows for time-travel queries in the warehouse, meeting the compliance and analysis requirements. This provides a detailed, auditable lineage of all transactions.


