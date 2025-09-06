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

**Why this design?**
 - **Time travel & ACID for analytics**: Snowflake provides built-in time-travel and easy point-in-time recovery — a direct match to the requirement for audits and debugging.
 - **Durable, cost-efficient landing store**: S3 holds raw immutable JSON and acts as the replayable source-of-truth.
 - **Low-latency ingestion**: Kinesis + Firehose + Snowpipe provides a managed, scalable ingestion path with near real-time behavior.
 - **Familiar BI & ML workflows**: Snowflake acts as an SQL-first warehouse for Tableau and BI; Databricks can access Snowflake for feature engineering and model development.
 - **Phased modernization**: Existing ETL (Informatica/SSIS) can be repointed to S3 or co-exist while migration proceeds.

## Additional Considerations

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
   
### Deduplication & Ordering
Kinesis + Firehose → S3 + Snowpipe can produce duplicated events (retries), partially-ordered data, and out-of-order delivery. When using CDC + event streams, duplicates or ordering issues break correctness (inventory, reconciliation).
 - **Mitigation**:
   - Adding a **unique event id** + **producer timestamp** to each event at the source or ingestion Lambda.
   - Dedupe layer: Snowflake staging step (raw → staging) that dedupes on (event_id, source).
   - For CDC from Oracle, DMS writes LSN/SCN (log sequence number) and operation type (INSERT/UPDATE/DELETE), reconcile ordering in Snowflake.

### Data Quality
- Potential JSON schema changes from Salesforce/Shopify:
   - Introducing a **Glue Schema Registry**
   - Validating incoming data at **Lambda** ingestion.
- **dbt tests** (data quality rules) via Streams/Tasks in Snowflake (uniqueness, not_null, accepted_values, relationships).

### Small Files & Cost Optimization
- Firehose can generate many small objects (especially at high partitioning), causing explosion in file counts and query inefficiency (Snowflake ingestion cost and later processing).
  - Tuning Firehose buffer_size/buffer_interval to produce optimally-sized files (e.g., 64–256MB compressed)
