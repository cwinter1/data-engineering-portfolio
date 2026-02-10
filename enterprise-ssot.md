# Enterprise Revenue SSOT & Semantic Layer

**Company:** Pepper Banking (Bank Leumi)  
**Role:** Sr. Data Analyst & BI Developer  
**Duration:** Nov 2021 – Nov 2024  
**Tech Stack:** PostgreSQL, Control-M, Tableau, Python, Custom Validation Framework

---

## 🎯 Business Challenge

The organization faced critical data reliability issues:

- **Conflicting KPIs:** Multiple teams calculated identical metrics with different results
- **Performance Issues:** Data pipelines took 6+ hours to complete, delaying business decisions
- **No Risk Analytics:** No systematic way to identify financially at-risk customers before losses
- **Dashboard Performance:** Heavy calculations at visualization layer causing slow load times
- **Trust Erosion:** Stakeholders questioning data accuracy and reliability

---

## 🏗️ Solution Architecture

### High-Level Design

```
┌─────────────────────────────────────────────────────────────────┐
│                      Source Systems                              │
│  PostgreSQL DBs • External APIs • Financial Systems • User Data  │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│              Centralized Semantic Layer (SSOT)                   │
│  • Standardized KPI Definitions                                  │
│  • Shared Dimensions & Facts                                     │
│  • Reusable Business Logic                                       │
│  • Data Lineage Tracking                                         │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│                   Control-M Orchestration                        │
│  • Automated Scheduling • Dependency Management • Error Handling │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│                    Analytics Layer                               │
│  Tableau Dashboards • Risk Models • Executive Reports            │
└─────────────────────────────────────────────────────────────────┘
```

---

## ⚙️ Technical Implementation

### 1. Semantic Layer Design

**Core Components:**

```sql
-- Centralized calculation engine with standardized definitions
CREATE VIEW semantic.revenue_metrics AS
SELECT 
    date_key,
    customer_id,
    product_type,
    -- Standardized revenue calculation
    SUM(transaction_amount) as total_revenue,
    -- Standardized customer lifetime value
    SUM(transaction_amount) / NULLIF(customer_tenure_months, 0) as monthly_clv,
    -- Risk indicators
    CASE 
        WHEN income_trend_3m < -0.15 THEN 'HIGH_RISK'
        WHEN income_trend_3m < -0.05 THEN 'MEDIUM_RISK'
        ELSE 'LOW_RISK'
    END as financial_risk_level
FROM fact_transactions
LEFT JOIN dim_customers USING (customer_id)
GROUP BY 1,2,3;
```

**Key Features:**
- ✅ Single definition for each KPI (no duplication)
- ✅ Shared dimensions (customer, product, time)
- ✅ Reusable calculation components
- ✅ Data lineage for audit trails

### 2. Performance Optimization

**Before:** 6+ hour processing time
**After:** 2.5 hours (58% improvement)

**Optimization Strategies:**

1. **Materialized Views**
```sql
CREATE MATERIALIZED VIEW mv_daily_revenue_summary AS
SELECT 
    date_key,
    product_type,
    customer_segment,
    SUM(revenue) as total_revenue,
    COUNT(DISTINCT customer_id) as unique_customers
FROM semantic.revenue_metrics
GROUP BY 1,2,3;

CREATE INDEX idx_mv_date_product ON mv_daily_revenue_summary(date_key, product_type);
```

2. **Query Optimization**
- Refactored joins (reduced from 12 to 6 tables)
- Implemented incremental processing
- Added strategic indexes on high-cardinality columns
- Partitioned fact tables by date

3. **Control-M Job Redesign**
- Parallelized independent workflows
- Optimized dependency chains
- Implemented smart scheduling (off-peak hours)

### 3. Risk Analytics Innovation

**Predictive Model for At-Risk Customers:**

```python
def calculate_customer_risk_score(customer_id):
    """
    Multi-factor risk assessment combining:
    - Income trend analysis (3-12 months)
    - Spending pattern changes
    - Payment behavior
    - Account activity
    """
    metrics = {
        'income_trend_3m': get_income_trend(customer_id, months=3),
        'income_trend_12m': get_income_trend(customer_id, months=12),
        'spending_volatility': calculate_spending_volatility(customer_id),
        'payment_delays': count_payment_delays(customer_id, months=6),
        'account_dormancy_days': get_dormancy_period(customer_id)
    }
    
    risk_score = (
        metrics['income_trend_3m'] * -2.0 +  # Declining income = risk
        metrics['spending_volatility'] * 1.5 +
        metrics['payment_delays'] * 3.0 +
        metrics['account_dormancy_days'] * 0.1
    )
    
    return risk_score, classify_risk(risk_score)
```

**Key Risk Indicators:**
- 📉 Income decline >15% over 3 months → HIGH RISK
- 💳 Spending pattern changes (unusual volatility)
- ⚠️ Payment delays or missed transactions
- 😴 Account inactivity periods

### 4. Data Quality Framework

**Automated Validation System:**

```python
class DataQualityValidator:
    """
    Comprehensive validation framework ensuring data integrity
    """
    
    def validate_revenue_data(self, date):
        checks = {
            'completeness': self.check_record_counts(date),
            'accuracy': self.validate_totals_reconciliation(date),
            'consistency': self.check_cross_table_consistency(date),
            'timeliness': self.verify_data_freshness(date),
            'integrity': self.validate_referential_integrity(date)
        }
        
        failed_checks = [k for k, v in checks.items() if not v['passed']]
        
        if failed_checks:
            self.send_alert(failed_checks)
            self.log_to_dashboard(checks)
        
        return all(v['passed'] for v in checks.values())
```

**Quality Metrics Tracked:**
- Primary/foreign key violations
- Null values in critical fields
- Data freshness (last update time)
- Reconciliation with source systems
- Historical trend anomalies

---

## 📈 Quantifiable Results

### Performance Improvements

| Metric | Before | After | Improvement |
|--------|--------|-------|-------------|
| **Processing Time** | 6 hours | 2.5 hours | **58% reduction** |
| **Dashboard Load Time** | 45 seconds | <1 second | **98% faster** |
| **Data Refresh Frequency** | Daily (morning) | Daily (evening) | **Same-day data** |
| **Pipeline Success Rate** | 94% | 99.5% | **Enhanced reliability** |

### Business Impact

- ✅ **100% KPI Consistency** - Eliminated conflicting calculations
- ✅ **15% of customer base** identified as at-risk before losses
- ✅ **Company-wide adoption** - Became official source of truth
- ✅ **Proactive interventions** - Enabled customer retention programs
- ✅ **Executive trust** - C-level decisions based on analytics

### Cost Savings

- **Reduced cloud compute:** 40% savings from optimized queries
- **Analyst productivity:** 50% less time on ad-hoc data validation
- **Infrastructure:** Avoided need for additional hardware

---

## 🎯 Key Learnings

### Technical Insights

1. **Semantic layers prevent data chaos**  
   Centralized definitions are worth the upfront investment

2. **Performance optimization compounds**  
   Small improvements (10-15%) at each layer = massive gains

3. **Automation is reliability**  
   Control-M orchestration eliminated manual intervention failures

4. **Trust requires proof**  
   Executive dashboards showing data quality metrics built confidence

### Business Insights

1. **Analytics should be proactive, not reactive**  
   Risk models prevented losses vs just reporting them

2. **Self-service needs guard rails**  
   SSOT enables independence without sacrificing accuracy

3. **Data governance is a competitive advantage**  
   Fast, reliable data = better decisions = business outcomes

---

## 🔄 Future Enhancements

If continuing this project, I would add:

- **Real-time streaming:** Kafka integration for sub-hourly updates
- **ML model integration:** Advanced predictive models in production
- **Data mesh architecture:** Decentralized ownership with centralized governance
- **DataOps automation:** Full CI/CD for data pipeline deployments
- **Observability:** Advanced monitoring with Datadog/Monte Carlo

---

## 🤝 Collaboration Approach

**Stakeholder Management:**
- Weekly syncs with business units to understand KPI needs
- Monthly reviews with C-level showing impact metrics
- Quarterly roadmap planning with IT and product teams

**Team Development:**
- Mentored 3 junior analysts on data modeling
- Created documentation and training materials
- Established best practices adopted company-wide

**Cross-functional Impact:**
- Partnered with product team on A/B testing framework
- Collaborated with finance on revenue recognition logic
- Worked with risk team on predictive model validation

---

## 📚 Technologies Used

**Data Storage & Processing:**
- PostgreSQL 12+ (materialized views, functions, triggers)
- Control-M (job orchestration)
- Python (data validation, risk models)

**Analytics & Visualization:**
- Tableau (dashboards, semantic layer integration)
- SQL (complex analytical queries)
- Custom validation framework

**DevOps & Governance:**
- Git (version control for SQL, Python)
- Data lineage tracking
- Automated testing

---

## 💡 Why This Project Matters

This wasn't just a technical implementation - it transformed how the organization made data-driven decisions:

- **From reactive to proactive:** Risk analytics enabled prevention, not just reporting
- **From siloed to unified:** Single source of truth broke down team barriers
- **From slow to fast:** 58% faster processing unlocked same-day insights
- **From questioned to trusted:** Quality framework built stakeholder confidence

**This is the kind of data infrastructure that enables business growth.**

---

[← Back to Portfolio](../README.md)
