# Solution Design Document
## Healthcare Data Analytics Platform

---

## 1. Solution Overview

### Objective
Increase revenue by analyzing customer behavior and sending targeted offers and royalties to customers.

### Business Problem
The healthcare company aims to enhance revenue and customer understanding by analyzing competitor data from multiple sources such as scraping and third-party datasets.

### Key Goals
- Track and understand customer behavior  
- Analyze claim trends  
- Identify royalty-eligible customers  
- Create targeted and customized offers  
- Identify profitable groups  
- Analyze spending patterns  

---

## 2. Use Cases

### 2.1 Analytics
- Understand policy hierarchy and group complexity  
- Perform customer segmentation  
- Analyze customer preferences for product design and marketing  
- Conduct geographic analysis (e.g., claims by city)  
- Study claim patterns, policy preferences, age groups, disease trends, and subgroup subscriptions  

### 2.2 Business Growth
- Identify profitable policy groups  
- Enable personalized insurance offers  
- Benchmark competitors (pricing, trends, coverage)  
- Implement loyalty/royalty programs  
- Improve cross-sell and up-sell opportunities  

### 2.3 Operations
- Improve data quality (cleaning, validation, standardization)  
- Automate data pipelines (ingestion → transformation → loading)  
- Track claim processing efficiency and rejection rates  
- Maintain centralized storage (S3 + Redshift)  
- Ensure scalability and reliability  
- Provide structured outputs for dashboards and reporting  

---

## 3. Database Design

### 3.1 Dimension Tables

#### Patient (`dim_patient`)
| Column Name     | Data Type | Key | Reference | Description |
|----------------|----------|-----|-----------|------------|
| patient_id     | bigint   | PK  |           | Unique patient ID |
| subscriber_id  | bigint   | FK  | dim_subscriber | Linked subscriber |
| patient_name   | varchar  |     |           | Patient name |
| gender         | varchar  |     |           | Gender |
| dob            | date     |     |           | Date of birth |
| age            | int      |     |           | Derived age |
| city           | varchar  |     |           | City |
| state          | varchar  |     |           | State |
| disease_name   | varchar  |     |           | Disease |
| admission_date | date     |     |           | Admission date |
| discharge_date | date     |     |           | Discharge date |
| procedure_name | varchar  |     |           | Procedure |

---

#### Subscriber (`dim_subscriber`)
| Column Name      | Data Type | Key | Reference | Description |
|------------------|----------|-----|-----------|------------|
| subscriber_id    | bigint   | PK  |           | Subscriber ID |
| subscriber_name  | varchar  |     |           | Name |
| dob              | date     |     |           | DOB |
| age              | int      |     |           | Age |
| gender           | varchar  |     |           | Gender |
| city             | varchar  |     |           | City |
| state            | varchar  |     |           | State |
| policy_id        | bigint   | FK  | dim_policy | Policy reference |
| group_id         | bigint   | FK  | dim_group | Group reference |
| subgroup_id      | bigint   | FK  | dim_subgroup | Subgroup reference |
| subscriber_type  | varchar  |     |           | Govt/Private |
| monthly_premium  | decimal  |     |           | Monthly premium |

---

#### Group (`dim_group`)
| Column Name | Data Type | Key | Description |
|-------------|----------|-----|------------|
| group_id    | bigint   | PK  | Group ID |
| group_name  | varchar  |     | Group name |
| group_type  | varchar  |     | Type |

---

#### Subgroup (`dim_subgroup`)
| Column Name   | Data Type | Key | Reference | Description |
|--------------|----------|-----|-----------|------------|
| subgroup_id  | bigint   | PK  |           | Subgroup ID |
| group_id     | bigint   | FK  | dim_group | Parent group |
| subgroup_name| varchar  |     |           | Subgroup name |

---

#### Hospital (`dim_hospital`)
| Column Name   | Data Type | Key | Description |
|--------------|----------|-----|------------|
| hospital_id  | bigint   | PK  | Hospital ID |
| hospital_name| varchar  |     | Hospital name |
| city         | varchar  |     | City |
| state        | varchar  |     | State |
| hospital_type| varchar  |     | Govt/Private |

---

#### Policy (`dim_policy`)
| Column Name   | Data Type | Key | Description |
|--------------|----------|-----|------------|
| policy_id    | bigint   | PK  | Policy ID |
| policy_name  | varchar  |     | Policy name |
| policy_type  | varchar  |     | Type |
| premium_amount | decimal|     | Premium |

---

### 3.2 Fact Tables

#### Claims (`fact_claim`)
| Column Name     | Data Type | Key | Reference | Description |
|----------------|----------|-----|-----------|------------|
| claim_id       | bigint   | PK  |           | Claim ID |
| patient_id     | bigint   | FK  | dim_patient | Patient |
| subscriber_id  | bigint   | FK  | dim_subscriber | Subscriber |
| policy_id      | bigint   | FK  | dim_policy | Policy |
| hospital_id    | bigint   | FK  | dim_hospital | Hospital |
| group_id       | bigint   | FK  | dim_group | Group |
| subgroup_id    | bigint   | FK  | dim_subgroup | Subgroup |
| claim_date     | date     |     |           | Claim date |
| claim_amount   | decimal  |     |           | Claim amount |
| approved_amount| decimal  |     |           | Approved amount |
| total_charges  | decimal  |     |           | Total charges |
| claim_status   | varchar  |     |           | Approved/Rejected |

---

#### Premium Payments (`fact_premium_payment`)
| Column Name   | Data Type | Key | Reference | Description |
|--------------|----------|-----|-----------|------------|
| payment_id   | bigint   | PK  |           | Payment ID |
| subscriber_id| bigint   | FK  | dim_subscriber | Subscriber |
| policy_id    | bigint   | FK  | dim_policy | Policy |
| payment_date | date     |     |           | Payment date |
| premium_amount | decimal|     |           | Amount paid |

---

## 4. KPI Design and Calculation

| KPI Name | Description | Source Tables | Logic | Output Table |
|----------|------------|--------------|-------|-------------|
| Claim Rejection Rate | % of rejected claims | claims | COUNT(rejected)/COUNT(*) | project_output.claim_rejection_rate |
| Total Premium Revenue | Total premium collected | subscriber | SUM(premium) | project_output.total_revenue |
| Top Disease by Claims | Most frequent disease | claims | GROUP BY disease ORDER BY COUNT DESC | project_output.top_disease |
| Most Profitable Group | Highest profit group | claims + subscriber + group | SUM(premium) - SUM(claim_amount) | project_output.profit_group |
| Average Claim Amount | Avg claim value | claims | AVG(claim_amount) | project_output.avg_claim |
| Claims by City | Claims distribution | claims + patient | GROUP BY city | project_output.claims_by_city |
| Subgroup Frequency | Most subscribed subgroup | subscriber + subgroup | COUNT GROUP BY subgroup | project_output.subgroup_frequency |
| Govt vs Private Ratio | Policy type distribution | subscriber | GROUP BY policy_type | project_output.policy_distribution |
| High Value Claims | Claims ≥ 50000 | claims | COUNT WHERE amount ≥ 50000 | project_output.high_value_claims |
| Top Hospital | Most patients served | hospital + patient | GROUP BY hospital ORDER BY COUNT DESC | project_output.top_hospital |

---

## 5. Technologies and Platforms

### 5.1 Architecture Layers

#### Layer 1: Data Lake
- AWS S3  
- Stores raw, staged, processed, and archived data  

#### Layer 2: Processing Layer
- PySpark  
- Databricks  
- AWS EMR  

Used for:
- Data ingestion  
- Cleaning & transformation  
- Deduplication  
- Aggregation  

#### Layer 3: Data Warehouse
- Amazon Redshift  

Used for:
- Curated datasets  
- Analytical queries  
- Reporting tables  

---

### 5.2 Data Modeling Approach
- Data Vault for core integration layer  
- Supports historical tracking and scalable modeling  

---

### 5.3 Detailed Technology Stack

#### Storage
- AWS S3 (raw, processed, intermediate, archive)

#### Processing
- PySpark (ETL, transformations, joins, aggregations)  
- Databricks (development, notebooks, collaboration)  
- AWS EMR (production execution)

#### Data Warehouse
- Redshift (analytics, reporting)

#### Supporting Tools
- GitHub (version control)  
- Jira (project management)  
- AWS Glue Catalog (metadata)  
- Apache Parquet (optimized storage format)  
- SQL (analytics queries)  
- Python (scripting)  
- CloudWatch (monitoring/logging)  
- IAM (security/access control)  
- Airflow / Databricks Workflows (orchestration)  

---
