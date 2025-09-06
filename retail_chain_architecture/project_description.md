# Data Lake Implementation for a Retail Chain
## Business Problem
XYZ Retail is a global retailer with both physical and e‑commerce operations. They need to address data‑integration and availability challenges caused by disconnected sources, slow reporting, and redundancy on‑prem data. Objectives include a centralized lake, real‑time ingestion, reduced reconciliation, cross‑channel insights, and self‑service BI.
## Current Ecosystem
 - **Warehouses**: Oracle, Teradata
 - **ETL**: Informatica PowerCenter, SSIS
 - **Reporting**: SAP BO, Tableau (decentralized)
 - **Processing**: Batch only (daily POS/ERP/web loads)
## Additional Technical & Business Requirements
1. **Current sources(API Based):** POS systems, Shopify‑like e‑commerce, Salesforce CRM, weather + pricing APIs
2. **Daily volume:** 5 – 10 million transaction rows (~2 TB/day)
3. **Data formats:** JSON (APIs)
4. **Real‑time feeds:** POS, e‑commerce orders
5. **Compliance:** PII (email, phone, address) → encryption, masking, RBAC, audit trails
6. **Platform preference:** AWS first choice; extensible to Azure/GCP
7. **Analytics tools:** Power BI, Tableau, Looker; SageMaker & Databricks for ML
8. **Growth (1‑3 yrs):** 3 – 5 × via new regions & IoT/mobile data
9. **Existing use‑cases:** abandoned‑cart recovery, store‑performance dash, inventory optimization, customer segmentation
10. **Time‑travel:** yes—audits, debugging, versioned datasets
11. **Primary users:** business analysts, inventory managers, data scientists, marketing analysts
12. **Success metrics:** report latency ↓ 24 h → 1 h; % self‑service reports; accuracy of real‑time inventory alerts; CSAT uplift
13. **API layer:** REST (CRM & e‑comm), optional GraphQL, Kafka REST Proxy for streaming

## Proposed Architecture
XYZ Retail will centralize streaming and batch sources into an **S3** landing zone with **Snowflake** as the canonical warehouse. Real-time sources (POS, Shopify-like e-commerce, Salesforce) stream through **Kinesis** and **Firehose** to **S3**; **S3** events via an **SQS queue** feed **Snowpipe**  which loads raw tables in **Snowflake**. **Snowflake** **Streams and Tasks** process the raw stream into staging and production tables for BI (Tableau/Power BI) and ML (Databricks/SageMaker). This design leverages **Snowflake** time-travel capabilities to meet audit/versioning requirements while maintaining **S3** as the immutable ground truth for portability and cost control.
<img width="1120" height="810" alt="retail_chain_diagram drawio" src="https://github.com/user-attachments/assets/67ccc8a1-edd2-4968-9e05-ca0c7bc94fc8" />

### PII Compliance
1. **Encryption**:
   - S3 → Server-Side Encryption with KMS (SSE-KMS).
   - Ingestion Lambda → use the KMS Encrypt API for field-level PII encryption.
   - Snowflake → supports AES-256 with customer-managed keys (Tri-Secret Secure).
2. **Masking** (prevents analysts from seeing decrypted PII if they have access)
   - In Snowflake: use Dynamic Data Masking so analysts see xxxxx@domain.com instead of full email.
3. **Role-Based Access Control (RBAC)**
   - Encryption key access is managed via KMS key policies + IAM roles (AWS) and role hierarchies (Snowflake).
   - Example:
     - Analysts → can query masked views only.
     - Data science team → can use encrypted tokens for segmentation.
     - Marketing ops → can call a Re-ID microservice that decrypts under strict approval.
