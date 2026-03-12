# Chris Winter - Data Engineering Portfolio

**Senior BI/Data Developer | Analytics Infrastructure & SSOT Architecture**

[![LinkedIn](https://img.shields.io/badge/LinkedIn-Connect-blue)](https://www.linkedin.com/in/chriswinter01/)
[![Email](https://img.shields.io/badge/Email-Contact-red)](mailto:new.chriswinter@gmail.com)

---

## 👋 About Me

Senior BI/Data Developer with 6+ years building production analytics infrastructure in FinTech and eCommerce. I focus on designing **Single Source of Truth (SSOT)** platforms — the kind that actually get adopted company-wide because people trust the numbers.

### Core expertise
- **SSOT & Semantic Layers** — standardizing KPIs across teams and tools
- **Performance Optimization** — 58% processing time reduction, 40% cost savings
- **Cloud Platforms** — AWS (S3/Glue/Athena) + GCP (BigQuery)
- **PostgreSQL** — advanced SQL, stored procedures, materialized views, performance tuning
- **Pipeline Orchestration** — Control-M, Airflow, automated scheduling

### 💼 Currently
**Sr. BI Developer & Analytics Engineer** at **Finubit (Bank Leumi Digital Banking)**
- Building production-grade data pipelines with Control-M
- Advanced PostgreSQL development and query optimization
- Integrating AI-assisted development into daily workflows

---

## 🚀 Projects

### 1️⃣ Enterprise Revenue SSOT & Semantic Layer
> Multiple teams calculating the same KPIs with different results. 6-hour pipeline runs. Stakeholders who didn't trust the data.

**What I built:**
- Centralized semantic layer with a single standardized calculation engine
- Control-M orchestration replacing fragmented, unreliable job scheduling
- Automated data validation and integrity checks at every pipeline stage
- Executive dashboards showing data reliability — not just business metrics

**Results:**
- ✅ **58% faster pipelines** (6 hours → 2.5 hours)
- ✅ **100% KPI consistency** across all business units
- ✅ **Company-wide adoption** — became the single source of truth
- ✅ **Proactive risk analytics** identifying at-risk customers before churn

**Stack:** PostgreSQL, Control-M, Tableau, Python

📂 [View Project Details](./projects/enterprise-ssot.md)

---

### 2️⃣ Production-Ready PostgreSQL Development Environment
> Needed a safe place to develop and test complex database logic without touching production.

**What I built:**
- Docker-based environment mirroring production database architecture
- Multi-schema design with FK constraints, custom functions, and stored procedures
- Role-based access control matching production security policies
- Python data access layer with connection pooling and a full pytest suite

**Results:**
- ✅ **Zero production risk** — fully isolated via Docker
- ✅ **2-minute setup** from scratch with a single command
- ✅ **Automated tests** covering FK integrity, view completeness, and function correctness
- ✅ **Consistent environments** across machines — no more "works on my machine"

**Stack:** Docker, PostgreSQL, Python, Git, Bash

📂 [View Project Details](./projects/postgresql-dev-environment.md)

---

### 3️⃣ Multi-Source Cloud Analytics Platform
> On-prem user data siloed from external cloud datasets — no infrastructure to combine them efficiently.

**What I built:**
- AWS Glue ETL pipelines for automated extraction and transformation
- Glue Data Catalog connecting PostgreSQL, S3, and Athena into a unified layer
- Cross-source joins combining on-prem behavioral data with external datasets
- Partitioned Parquet storage for cost-efficient serverless querying

**Results:**
- ✅ **40% infrastructure cost reduction** via serverless architecture
- ✅ **5+ data sources** consolidated into one reporting framework
- ✅ **Hours → minutes** for cross-source report generation
- ✅ **Scalable by design** — handles growing volumes without re-engineering

**Stack:** AWS (S3, Glue, Athena), BigQuery, PostgreSQL, Python

📂 [View Project Details](./projects/cloud-analytics.md)

---

### 4️⃣ High-Performance Materialized Views with Automated Scheduling
> Report queries running 10+ minutes during business hours, putting load on production systems.

**What I built:**
- Optimized materialized view replacing complex multi-join queries with pre-aggregated results
- Composite indexes designed around actual dashboard access patterns
- CONCURRENTLY refresh so dashboards stay live during the refresh window
- Control-M job chain with upstream dependencies, SLA monitoring, and failure alerting

**Results:**
- ✅ **>95% query time reduction** (10+ minutes → sub-second)
- ✅ **Zero production load** from analytical queries during business hours
- ✅ **99.5% refresh reliability** with automated error recovery
- ✅ **Predictable availability** — dashboards ready every morning without manual intervention

**Stack:** PostgreSQL, Control-M, SQL

📂 [View Project Details](./projects/materialized-views.md)

---

### 5️⃣ Product Funnel Analytics & User Journey Optimization
> High drop-off rates with no visibility into where users were failing or why.

**What I built:**
- BigQuery event processing pipeline covering web and mobile user actions
- Dual dashboard system: happy flow (conversions) vs error flow (failures and abandonment)
- Cohort analysis and A/B test integration for measuring product changes
- Automated alerting for unusual drop-off patterns before they compound

**Results:**
- ✅ **23% conversion rate improvement** through targeted funnel fixes
- ✅ **Days → hours** for identifying critical errors
- ✅ **40% faster product iterations** with data-backed decisions
- ✅ **15% fewer support tickets** related to flow completion

**Stack:** BigQuery, SQL, Looker, Python

📂 [View Project Details](./projects/product-analytics.md)

---

### 6️⃣ AI-Assisted Development Infrastructure
> Manual code generation, slow iteration cycles, documentation overhead eating into development time.

**What I built:**
- Claude CLI integrated into daily development workflow
- Automated query optimization and code review pipeline
- Documentation generation tied to Git commits
- Team adoption framework with validation standards

**Results:**
- ✅ **40% faster development** cycles
- ✅ **Quality maintained** through automated validation
- ✅ **Team-wide adoption** of AI-assisted workflows
- ✅ **Automated testing** integrated with Git/GitHub

**Stack:** Claude CLI, Python, Git/GitHub, Docker, CI/CD

📂 [View Project Details](./projects/ai-development.md)

---

## 🛠️ Technical Skills

### Databases & Analytics
```
PostgreSQL (Advanced)  •  SQL (Expert)  •  BigQuery  •  AWS Athena
Stored Procedures  •  Materialized Views  •  Window Functions  •  Query Optimization
```

### Cloud & ETL
```
AWS (S3, Glue, Athena)  •  GCP (BigQuery)
AWS Glue  •  Python Pipelines  •  Data Lake Architecture
```

### Orchestration & DevOps
```
Control-M  •  Airflow  •  Git/GitHub  •  Docker  •  CI/CD
```

### BI & Visualization
```
Tableau (Advanced)  •  Looker  •  Power BI  •  Grafana
Semantic Modeling  •  Funnel Analysis  •  A/B Testing
```

---

## 📊 Impact by the numbers

| Metric | Result |
|--------|--------|
| Pipeline processing time | 58% faster (6hrs → 2.5hrs) |
| Infrastructure cost | 40% reduction |
| Conversion rate | 23% improvement |
| Development speed | 40% faster with AI assistance |
| Pipeline reliability | 99.5% uptime |
| KPIs standardized | 100+ across all business units |

---

## 📈 Experience

### **Finubit (Bank Leumi Digital Banking)** — Dec 2024 – Present
*Sr. BI Developer & Analytics Engineer*
- Production data pipelines with Control-M orchestration
- Advanced PostgreSQL development and optimization
- AI-assisted development workflows
- DevOps & CI/CD implementation

### **Pepper Banking (Bank Leumi)** — Nov 2021 – Nov 2024
*Sr. Data Analyst & BI Developer*
- Enterprise SSOT architecture and semantic layer design
- Multi-cloud platform integration (AWS + GCP)
- Product analytics and revenue optimization
- Predictive risk modeling

### **TCM eCommerce** — May 2019 – Oct 2021
*BI Team Lead & Analyst*
- Led team building end-to-end BI solutions
- Designed scalable data warehouse
- Built self-service BI platform

### **DSNR** — Nov 2018 – Apr 2019
*Data Analyst*
- KPI monitoring dashboards
- Performance marketing analytics

---

## 🎓 Education

**Bachelor's Degree in Business Administration**
Ruppin Academic Center | 2008 – 2011

---

## 📫 Get in touch

- Email: new.chriswinter@gmail.com
- Phone: 052-8924757
- LinkedIn: [linkedin.com/in/chriswinter01](https://www.linkedin.com/in/chriswinter01/)
- Location: Netanya, Israel

---

## 🌍 Languages

Hebrew (Native) • English (Native) • Spanish (Native)

German citizenship available

---

*Last updated: March 2026*
