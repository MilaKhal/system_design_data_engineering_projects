# Data Governance & Data Quality Framework for a Healthcare Company
## Business Problem
DEF Healthcare faces duplicate patient records and inconsistent diagnoses due to poor governance. Needs MDM, standardized quality, privacy controls, and role‑based access.
## Current Ecosystem
 - **EHRs**: EHRs: Cerner, Epic, Meditech
 - **Regional marts**: SQL Server
 - **Cleanup**: Excel manual
## Additional Technical & Business Requirements
1. **Current sources(API Based):** POS systems, Shopify‑like e‑commerce, Salesforce CRM, weather + pricing APIs
2. **Daily volume:** 5 – 10 million transaction rows (~2 TB/day)
3. **Data formats:** JSON (APIs)
4. **Real‑time feeds:** POS, e‑commerce orders
5. **Compliance:** PII (email, phone, address) → encryption, masking, RBAC, audit (HIPAA, GDPR)
6. **Platform preference:** AWS first choice; extensible to Azure/GCP
7. **Analytics tools:** Power BI, Tableau, Looker; SageMaker & Databricks for ML
8. **Growth (1‑3 yrs):** 3 – 5 × via new regions & IoT/mobile data
9. **Existing use‑cases:** abandoned‑cart recovery, store‑performance dash, inventory optimization, segmentation (plus unified patient record, readmission prediction)
10. **Time‑travel:** yes—audits, debugging, versioned datasets
11. **Primary users:** clinicians, data stewards, compliance officers, data scientists
12. **Success metrics:** duplicate rate ↓, report latency, % compliant access, audit‑pass rate
13. **API layer:** REST (CRM & e‑comm), optional GraphQL, Kafka REST Proxy for streaming

## Proposed Architecture
<img width="3840" height="2083" alt="Untitled diagram _ Mermaid Chart-2025-09-10-002913" src="https://github.com/user-attachments/assets/945e9710-4ea6-4245-ae94-f8917ea81133" />

- **Data Ingestion:** Raw data from **EHRs (Cerner, Epic, Meditech)** and new sources (APIs, IoT, tele-health) is ingested in various formats **(JSON, CSV, Avro)**. This real-time ingestion, handling 5-10 million rows/day, is managed by **Kafka**.
  
 - **Data Lake:** Data is stored in a scalable, cost-effective data lake ( **Amazon S3**) in open formats like **Parque**t and **ORC**. This allows for time-travel queries and serves as the single source of truth.
- **Data Quality (DQ) Engine**: A DQ engine (e.g., **AWS Glue Data Quality, Great Expectations**) profiles and validates the ingested data. It identifies and flags issues like malformed entries or inconsistent formats.
- **Master Data Management (MDM) Hub:** The **MDM hub**(e.g., **Informatica MDM, Reltio**) is the core of the system. It ingests data from the DQ engine, applies survivorship rules, and links disparate patient records (e.g., identifying a patient with multiple records from Cerner and Epic as a single entity). This process directly addresses the issue of duplicate patient records.
- **Data Warehouse/Marts:** Cleaned and unified data from the MDM hub is loaded into a data warehouse (e.g., **Amazon Redshift**) or regional data marts (**SQL Server**). This provides a structured, high-performance environment for reporting and analysis.
- **Data Lineage:** Tools like **AWS Glue Data Catalog** or dedicated lineage platforms track the data's journey from source to destination. This shows how a specific diagnosis or patient record was created and transformed, providing transparency and supporting audit trails. This addresses the need for data lineage.
- **Access Control & Privacy:** This layer is critical for **PHI + PII** protection.
   - **Encryption:** Data is encrypted both at rest (e.g., S3 server-side encryption) and in transit (TLS/SSL).
   - **Masking:** PII is masked for non-production environments and for roles that don't require access to sensitive information.
   - **Role-Based Access Control (RBAC):** Fine-grained access policies are enforced to ensure that clinicians, data scientists, compliance officers, and data stewards only access the data required for their roles. For example, a data scientist may see aggregated, anonymized data, while a clinician can view specific patient records. This directly addresses the need for RBAC/privacy controls.
- **Analytics & ML:** Cleansed and unified data is consumed by tools like **Power BI, Tableau, and Looker** for dashboards and reporting. **SageMaker** and **Databricks** use this data for predictive models, such as readmission prediction.
- **Success Monitoring:** Metrics like duplicate rate, report latency, compliant access, and audit-pass rate are monitored to ensure the framework's success.

## Benefits
- **Single Source of Truth**: The **MDM hub** creates a unified patient record, eliminating data silos and inconsistent diagnoses caused by fragmented data. This provides a holistic view of the patient for clinicians and data scientists.
- **Improved Data Quality:** The **DQ engine** and standardized processes ensure data is clean, accurate, and reliable, leading to more trustworthy reports and better-performing machine learning models.
- **Enhanced Compliance:** Strong **RBAC**, encryption, and audit trails ensure adherence to regulations like **HIPAA and GDPR**. This significantly improves the audit-pass rate.
- **Operational Efficiency:** Automation of data cleanup and integration reduces the reliance on manual **Excel cleanup**, freeing up resources and reducing human error.
- **Scalability:** The **AWS-based architecture** can handle the 3-5x growth from new sources like tele-health and IoT, ensuring the framework remains effective as the business expands.

## Risk Mitigations
- **Duplicate Records & Inconsistent Data**: The **MDM hub** uses sophisticated matching and survivorship rules to link duplicate records, directly addressing the core problem. The DQ engine validates incoming data, preventing bad data from entering the system.
- **Privacy & Security Breaches:**
    - **Encryption** mitigates the risk of data exposure.
    - **RBAC** policies ensure only authorized users can access sensitive PHI/PII.
    - **Audit trails** provide a detailed record of data access, allowing for rapid detection and response to unauthorized activity.
- **Data Silos:** The data lake serves as a central repository, breaking down silos and providing a unified data source for all downstream systems and users.
- **Integration Complexity:** Adopting a robust **MDM hub** and a standardized ingestion layer with Kafka simplifies the process of integrating new data sources like APIs and new EHRs, reducing the risk of a messy, manual integration process.
- **Lack of Governance:** Defining clear roles for data stewards and compliance officers and establishing a formal Data Governance Committee ensures accountability and ongoing maintenance of the framework.
