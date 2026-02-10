# Product Funnel Analytics & Revenue Optimization

**Company:** Pepper Banking (Bank Leumi)  
**Role:** Sr. Data Analyst & BI Developer  
**Duration:** Nov 2021 – Nov 2024  
**Tech Stack:** BigQuery, SQL, Looker, Python, Automated Monitoring

---

## 🎯 Business Challenge

The product team struggled with high drop-off rates but lacked critical insights:

- **Where users dropped off** in multi-step processes was unknown
- **Why users failed** to complete actions couldn't be diagnosed
- **UX issues vs technical errors** were indistinguishable
- **Impact of product changes** on conversion was unmeasured
- **Reactive approach** - discovering problems after they affected many users

---

## 🏗️ Solution Architecture

### System Design

```
┌──────────────────────────────────────────────────────────────┐
│                      Event Sources                            │
│  Web Events  •  Mobile Events  •  API Calls  •  Page Views   │
└────────────────────────┬─────────────────────────────────────┘
                         │
                         ▼
┌──────────────────────────────────────────────────────────────┐
│                   BigQuery Data Warehouse                     │
│  • Event Processing  • User Journey Tracking                 │
│  • Stage Definitions • Cohort Analysis                       │
└────────────────────────┬─────────────────────────────────────┘
                         │
         ┌───────────────┼───────────────┐
         ▼               ▼               ▼
┌──────────────┐  ┌─────────────┐  ┌──────────────┐
│ Happy Flow   │  │ Error Flow  │  │ Real-Time    │
│ Dashboard    │  │ Dashboard   │  │ Alerts       │
│              │  │             │  │              │
│ • Success    │  │ • Failures  │  │ • Anomalies  │
│ • Completion │  │ • Drop-offs │  │ • Incidents  │
│ • Engagement │  │ • Issues    │  │ • Monitoring │
└──────────────┘  └─────────────┘  └──────────────┘
```

---

## ⚙️ Technical Implementation

### 1. Event Tracking Infrastructure

**BigQuery Event Schema:**

```sql
CREATE TABLE analytics.user_events (
    event_id STRING NOT NULL,
    event_timestamp TIMESTAMP NOT NULL,
    user_id STRING NOT NULL,
    session_id STRING NOT NULL,
    event_name STRING NOT NULL,
    event_properties JSON,
    
    -- Page context
    page_url STRING,
    referrer_url STRING,
    
    -- User context
    device_type STRING,
    browser STRING,
    country_code STRING,
    
    -- Funnel tracking
    funnel_name STRING,
    funnel_step INT64,
    funnel_step_name STRING,
    step_completed BOOLEAN,
    
    -- Error tracking
    error_occurred BOOLEAN,
    error_type STRING,
    error_message STRING,
    
    -- Performance
    page_load_time_ms INT64,
    api_response_time_ms INT64
)
PARTITION BY DATE(event_timestamp)
CLUSTER BY user_id, funnel_name;
```

### 2. Funnel Analysis Framework

**Stage Tracking Logic:**

```sql
-- Define multi-step product funnel with completion criteria
WITH funnel_stages AS (
    SELECT 
        user_id,
        session_id,
        
        -- Stage 1: Landing
        MIN(CASE WHEN event_name = 'page_view' 
                  AND page_url LIKE '%/signup%' 
             THEN event_timestamp END) as stage_1_landing,
        
        -- Stage 2: Form Started
        MIN(CASE WHEN event_name = 'form_started' 
             THEN event_timestamp END) as stage_2_form_start,
        
        -- Stage 3: Personal Info
        MIN(CASE WHEN event_name = 'form_section_completed' 
                  AND JSON_EXTRACT_SCALAR(event_properties, '$.section') = 'personal_info'
             THEN event_timestamp END) as stage_3_personal,
        
        -- Stage 4: Verification
        MIN(CASE WHEN event_name = 'verification_initiated'
             THEN event_timestamp END) as stage_4_verification,
        
        -- Stage 5: Completion
        MIN(CASE WHEN event_name = 'signup_completed'
             THEN event_timestamp END) as stage_5_completion
    
    FROM analytics.user_events
    WHERE DATE(event_timestamp) >= CURRENT_DATE() - 7
    AND funnel_name = 'user_signup'
    GROUP BY 1, 2
),

-- Calculate conversion rates between stages
funnel_conversion AS (
    SELECT
        COUNT(DISTINCT user_id) as users_entered,
        COUNT(DISTINCT CASE WHEN stage_2_form_start IS NOT NULL THEN user_id END) as stage_2_users,
        COUNT(DISTINCT CASE WHEN stage_3_personal IS NOT NULL THEN user_id END) as stage_3_users,
        COUNT(DISTINCT CASE WHEN stage_4_verification IS NOT NULL THEN user_id END) as stage_4_users,
        COUNT(DISTINCT CASE WHEN stage_5_completion IS NOT NULL THEN user_id END) as completed_users,
        
        -- Conversion rates
        SAFE_DIVIDE(
            COUNT(DISTINCT CASE WHEN stage_2_form_start IS NOT NULL THEN user_id END),
            COUNT(DISTINCT user_id)
        ) * 100 as stage_1_to_2_conversion,
        
        SAFE_DIVIDE(
            COUNT(DISTINCT CASE WHEN stage_5_completion IS NOT NULL THEN user_id END),
            COUNT(DISTINCT user_id)
        ) * 100 as overall_conversion
        
    FROM funnel_stages
)

SELECT * FROM funnel_conversion;
```

### 3. Dual Dashboard Strategy

**Happy Flow Dashboard (Success Metrics):**

```sql
-- Track successful user journeys and positive indicators
CREATE OR REPLACE VIEW analytics.happy_flow_metrics AS
SELECT
    DATE(event_timestamp) as date,
    funnel_name,
    
    -- Success metrics
    COUNT(DISTINCT CASE WHEN step_completed = TRUE THEN user_id END) as users_completed,
    AVG(CASE WHEN step_completed = TRUE THEN page_load_time_ms END) as avg_load_time_success,
    
    -- Completion time analysis
    APPROX_QUANTILES(
        TIMESTAMP_DIFF(completion_time, start_time, SECOND), 
        100
    )[OFFSET(50)] as median_time_to_complete,
    
    -- Feature adoption
    COUNT(DISTINCT CASE WHEN event_name LIKE '%feature_used%' THEN user_id END) as feature_adopters,
    
    -- Retention indicators
    COUNT(DISTINCT CASE WHEN return_visit = TRUE THEN user_id END) as returning_users

FROM analytics.user_events
WHERE step_completed = TRUE
GROUP BY 1, 2;
```

**Error Flow Dashboard (Failure Analysis):**

```sql
-- Track failures, errors, and drop-off points
CREATE OR REPLACE VIEW analytics.error_flow_metrics AS
SELECT
    DATE(event_timestamp) as date,
    funnel_name,
    funnel_step_name,
    
    -- Error categorization
    error_type,
    COUNT(*) as error_count,
    COUNT(DISTINCT user_id) as users_affected,
    
    -- Error patterns
    COUNT(DISTINCT session_id) as sessions_with_errors,
    AVG(CASE WHEN error_occurred THEN api_response_time_ms END) as avg_response_time_on_error,
    
    -- Drop-off analysis
    COUNT(DISTINCT CASE 
        WHEN error_occurred 
        AND NOT EXISTS (
            SELECT 1 FROM analytics.user_events e2
            WHERE e2.user_id = user_events.user_id
            AND e2.event_timestamp > user_events.event_timestamp
            AND e2.funnel_name = user_events.funnel_name
        )
        THEN user_id 
    END) as users_abandoned_after_error,
    
    -- Recovery patterns
    COUNT(DISTINCT CASE
        WHEN error_occurred
        AND EXISTS (
            SELECT 1 FROM analytics.user_events e2
            WHERE e2.user_id = user_events.user_id
            AND e2.event_timestamp > user_events.event_timestamp
            AND e2.step_completed = TRUE
        )
        THEN user_id
    END) as users_recovered

FROM analytics.user_events
WHERE error_occurred = TRUE
GROUP BY 1, 2, 3, 4;
```

### 4. Advanced Segmentation

**Cohort Analysis:**

```python
# cohort_analysis.py
import pandas as pd
from google.cloud import bigquery

class CohortAnalyzer:
    """
    Analyze user behavior by cohort for retention and conversion patterns
    """
    
    def __init__(self, project_id):
        self.client = bigquery.Client(project=project_id)
    
    def calculate_cohort_retention(self, funnel_name, cohort_period='month'):
        """
        Track how different user cohorts convert and retain over time
        """
        query = f"""
        WITH user_cohorts AS (
            SELECT
                user_id,
                DATE_TRUNC(MIN(event_timestamp), {cohort_period.upper()}) as cohort_date,
                MIN(event_timestamp) as first_event
            FROM analytics.user_events
            WHERE funnel_name = '{funnel_name}'
            GROUP BY user_id
        ),
        cohort_activity AS (
            SELECT
                c.cohort_date,
                c.user_id,
                DATE_DIFF(
                    DATE(e.event_timestamp),
                    DATE(c.first_event),
                    DAY
                ) as days_since_first,
                MAX(e.step_completed) as completed_in_period
            FROM user_cohorts c
            JOIN analytics.user_events e USING (user_id)
            WHERE e.funnel_name = '{funnel_name}'
            GROUP BY 1, 2, 3
        )
        SELECT
            cohort_date,
            COUNT(DISTINCT user_id) as cohort_size,
            COUNT(DISTINCT CASE WHEN days_since_first <= 1 THEN user_id END) as day_1_active,
            COUNT(DISTINCT CASE WHEN days_since_first <= 7 THEN user_id END) as day_7_active,
            COUNT(DISTINCT CASE WHEN days_since_first <= 30 THEN user_id END) as day_30_active,
            COUNT(DISTINCT CASE WHEN completed_in_period THEN user_id END) as completed_users
        FROM cohort_activity
        GROUP BY cohort_date
        ORDER BY cohort_date;
        """
        
        return self.client.query(query).to_dataframe()
```

### 5. Real-Time Alerting

**Anomaly Detection System:**

```python
# anomaly_detection.py
import numpy as np
from datetime import datetime, timedelta

class FunnelAnomalyDetector:
    """
    Detect unusual patterns in funnel metrics for proactive incident response
    """
    
    def check_conversion_anomaly(self, funnel_name, current_rate):
        """
        Alert if conversion rate deviates significantly from baseline
        """
        # Get historical baseline (30-day average)
        query = f"""
        SELECT
            AVG(conversion_rate) as baseline_rate,
            STDDEV(conversion_rate) as std_dev
        FROM (
            SELECT
                DATE(event_timestamp) as date,
                COUNT(DISTINCT CASE WHEN step_completed THEN user_id END) /
                COUNT(DISTINCT user_id) as conversion_rate
            FROM analytics.user_events
            WHERE funnel_name = '{funnel_name}'
            AND event_timestamp >= CURRENT_TIMESTAMP() - INTERVAL 30 DAY
            GROUP BY date
        )
        """
        
        result = self.run_query(query)
        baseline = result['baseline_rate'][0]
        std_dev = result['std_dev'][0]
        
        # Alert if more than 2 standard deviations below baseline
        z_score = (current_rate - baseline) / std_dev
        
        if z_score < -2:
            self.send_alert(
                severity='HIGH',
                message=f'{funnel_name} conversion dropped {abs(z_score):.1f}σ below baseline',
                current_rate=current_rate,
                baseline_rate=baseline
            )
            return True
        
        return False
    
    def check_error_spike(self, funnel_name, time_window_minutes=15):
        """
        Alert on sudden increase in error rates
        """
        query = f"""
        SELECT
            COUNT(CASE WHEN error_occurred THEN 1 END) as errors,
            COUNT(*) as total_events,
            SAFE_DIVIDE(
                COUNT(CASE WHEN error_occurred THEN 1 END),
                COUNT(*)
            ) as error_rate
        FROM analytics.user_events
        WHERE funnel_name = '{funnel_name}'
        AND event_timestamp >= CURRENT_TIMESTAMP() - INTERVAL {time_window_minutes} MINUTE
        """
        
        result = self.run_query(query)
        
        # Alert if error rate exceeds 5%
        if result['error_rate'][0] > 0.05:
            self.send_alert(
                severity='CRITICAL',
                message=f'{funnel_name} error rate at {result["error_rate"][0]*100:.1f}%',
                errors=result['errors'][0],
                total_events=result['total_events'][0]
            )
```

---

## 📈 Quantifiable Results

### Conversion Improvements

| Funnel | Before | After | Improvement |
|--------|--------|-------|-------------|
| **User Signup** | 12.3% | 15.1% | **+23% conversion** |
| **Loan Application** | 8.7% | 10.2% | **+17% conversion** |
| **Credit Card** | 15.4% | 18.2% | **+18% conversion** |

### Operational Impact

- ✅ **Incident detection time:** 2 hours → 5 minutes (96% faster)
- ✅ **Support ticket reduction:** 15% decrease (issues caught proactively)
- ✅ **Product iteration speed:** 40% faster (data-driven decisions)
- ✅ **Error resolution:** Average 30 minutes to identify root cause

### Business Value

- **Revenue impact:** 23% conversion improvement = significant ARR growth
- **User experience:** Faster issue resolution = better retention
- **Team efficiency:** Product team self-service reduced analyst requests 60%
- **Proactive management:** Catching issues before they scale

---

## 🎯 Key Learnings

### Technical Insights

1. **Dual dashboards are essential**  
   Success metrics AND failure metrics tell the complete story

2. **Real-time matters for UX**  
   Batch analysis is too slow - need <15min detection for critical issues

3. **Segmentation reveals patterns**  
   Different user cohorts behave differently - aggregate metrics hide insights

4. **Error categorization is critical**  
   Technical errors vs UX confusion require different solutions

### Business Insights

1. **Small improvements compound**  
   5% conversion gain per funnel = millions in revenue

2. **Proactive beats reactive**  
   Catching drops instantly prevents large user impact

3. **Data-driven product development works**  
   Teams iterate faster with clear metrics

4. **User journey mapping is competitive advantage**  
   Understanding where and why users fail enables optimization

---

## 🔄 Future Enhancements

Next steps for this project:

- **Predictive analytics:** ML models forecasting conversion likelihood
- **Automated experimentation:** A/B testing framework with auto-rollback
- **Session replay:** Video recordings of error sessions
- **Advanced attribution:** Multi-touch attribution across channels
- **Personalization:** Dynamic funnel optimization per user segment

---

## 📚 Technologies Used

**Data Warehouse & Processing:**
- BigQuery (event storage, analytics)
- SQL (funnel analysis, metrics calculation)
- Python (anomaly detection, cohort analysis)

**Visualization & Monitoring:**
- Looker (dashboards, self-service)
- Custom alerting system
- Slack integration (real-time notifications)

**Analytics Framework:**
- Event tracking schema design
- Funnel methodology
- Cohort analysis framework

---

## 💡 Why This Project Matters

This analytics platform transformed product development from guesswork to science:

- **From reactive to proactive:** Catching issues in real-time
- **From aggregate to granular:** Understanding user-level behavior
- **From slow to fast:** Immediate feedback on product changes
- **From opinions to data:** Metrics-driven prioritization

**Result:** 23% conversion improvement = measurable business growth

---

[← Back to Portfolio](../README.md)
