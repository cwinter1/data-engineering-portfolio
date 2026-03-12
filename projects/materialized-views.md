# High-Performance Materialized Views with Automated Scheduling

**Company:** FinTech/Banking
**Role:** Senior BI / Data Developer
**Tech Stack:** PostgreSQL, Control-M, SQL Optimization, Data Modeling

---

## 🎯 Business Challenge

Daily data processing involved massive datasets that caused cascading problems:

- **Performance Issues:** Report queries taking 10+ minutes during business hours
- **Resource Contention:** Heavy analytical queries degrading production system performance
- **Inconsistent Availability:** Ad-hoc data refreshes caused unpredictable report delays
- **Scalability Concerns:** Growing data volumes threatening system stability

---

## 🏗️ Solution Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                      Source Tables                               │
│  transactions • customers • accounts • external_feeds            │
└────────────────────────┬────────────────────────────────────────┘
                         │  Multiple JOINs, heavy aggregations
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│              Optimized Materialized View                         │
│  • Pre-aggregated KPIs                                          │
│  • Denormalized for read performance                            │
│  • Composite indexes on access patterns                         │
│  • Single optimized table replacing complex queries             │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│                  Control-M Scheduler                             │
│  • Daily refresh at 18:30 (after source data complete)          │
│  • Job dependency chains for consistency                        │
│  • Failure detection and alerting                               │
│  • Monitoring of refresh duration and success rate              │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│                    BI Dashboards                                 │
│  Sub-second response • Consistent availability • No load impact  │
└─────────────────────────────────────────────────────────────────┘
```

---

## ⚙️ Technical Implementation

### 1. Materialized View Design

**Before — Query hitting source tables directly:**

```sql
-- Original report query: 10+ minutes
SELECT
    c.customer_id,
    c.segment,
    c.risk_tier,
    COUNT(t.transaction_id)                                     AS total_transactions,
    SUM(t.amount)                                               AS total_spend,
    SUM(t.amount) FILTER (WHERE t.transaction_date >= CURRENT_DATE - 30) AS spend_30d,
    SUM(t.amount) FILTER (WHERE t.transaction_date >= CURRENT_DATE - 90) AS spend_90d,
    AVG(t.amount)                                               AS avg_transaction,
    MAX(t.transaction_date)                                     AS last_activity,
    COUNT(DISTINCT DATE_TRUNC('month', t.transaction_date))     AS active_months,
    SUM(e.exposure_amount)                                      AS total_exposure,
    MAX(r.risk_score)                                           AS current_risk_score
FROM customers c
LEFT JOIN transactions t          USING (customer_id)
LEFT JOIN credit_exposure e       USING (customer_id)
LEFT JOIN risk_assessments r      ON c.customer_id = r.customer_id
                                 AND r.assessment_date = CURRENT_DATE
GROUP BY c.customer_id, c.segment, c.risk_tier;
```

**After — Optimized materialized view:**

```sql
-- Materialized view: engineered for common query patterns
CREATE MATERIALIZED VIEW analytics.customer_kpi_summary AS
SELECT
    c.customer_id,
    c.segment,
    c.risk_tier,
    c.onboarding_date,

    -- Transaction aggregates
    COUNT(t.transaction_id)                                              AS total_transactions,
    SUM(t.amount)                                                        AS total_spend,
    AVG(t.amount)                                                        AS avg_transaction,
    MAX(t.transaction_date)                                              AS last_activity_date,
    COUNT(DISTINCT DATE_TRUNC('month', t.transaction_date))              AS active_months,

    -- Rolling windows — pre-calculated at refresh time
    SUM(t.amount) FILTER (WHERE t.transaction_date >= CURRENT_DATE - 30) AS spend_30d,
    SUM(t.amount) FILTER (WHERE t.transaction_date >= CURRENT_DATE - 90) AS spend_90d,
    COUNT(t.transaction_id) FILTER (
        WHERE t.transaction_date >= CURRENT_DATE - 30
    )                                                                    AS txn_count_30d,

    -- Income / spend trend (3-month vs prior 3-month)
    SUM(t.amount) FILTER (
        WHERE t.transaction_date BETWEEN CURRENT_DATE - 90 AND CURRENT_DATE - 60
    )                                                                    AS spend_90_60d,

    -- Risk snapshot
    MAX(r.risk_score)                                                    AS current_risk_score,
    SUM(e.exposure_amount)                                               AS total_exposure,

    -- Metadata
    NOW()                                                                AS refreshed_at

FROM customers c
LEFT JOIN transactions t     USING (customer_id)
LEFT JOIN credit_exposure e  USING (customer_id)
LEFT JOIN risk_assessments r ON c.customer_id = r.customer_id
                             AND r.assessment_date = CURRENT_DATE
GROUP BY c.customer_id, c.segment, c.risk_tier, c.onboarding_date;
```

### 2. Indexing Strategy

```sql
-- Unique index required for CONCURRENTLY refresh
CREATE UNIQUE INDEX idx_kpi_customer_id
    ON analytics.customer_kpi_summary (customer_id);

-- Composite indexes matching primary dashboard filter patterns
CREATE INDEX idx_kpi_segment_risk
    ON analytics.customer_kpi_summary (segment, risk_tier);

CREATE INDEX idx_kpi_risk_score
    ON analytics.customer_kpi_summary (current_risk_score DESC)
    WHERE current_risk_score IS NOT NULL;

CREATE INDEX idx_kpi_last_activity
    ON analytics.customer_kpi_summary (last_activity_date DESC);

-- Partial index for at-risk customers (small, fast)
CREATE INDEX idx_kpi_high_risk
    ON analytics.customer_kpi_summary (customer_id, total_exposure)
    WHERE current_risk_score >= 70;
```

### 3. Refresh Procedure with Monitoring

```sql
CREATE OR REPLACE PROCEDURE analytics.refresh_kpi_summary()
LANGUAGE plpgsql AS $$
DECLARE
    v_start         TIMESTAMP := NOW();
    v_row_count     BIGINT;
    v_duration_sec  NUMERIC;
BEGIN
    -- Non-blocking refresh (requires unique index)
    REFRESH MATERIALIZED VIEW CONCURRENTLY analytics.customer_kpi_summary;

    SELECT COUNT(*) INTO v_row_count FROM analytics.customer_kpi_summary;
    v_duration_sec := EXTRACT(EPOCH FROM NOW() - v_start);

    -- Log successful refresh
    INSERT INTO audit.mv_refresh_log (
        view_name, run_at, duration_seconds, row_count, status
    ) VALUES (
        'customer_kpi_summary', v_start, v_duration_sec, v_row_count, 'success'
    );

    -- Alert if refresh took longer than SLA (45 min)
    IF v_duration_sec > 2700 THEN
        INSERT INTO audit.mv_refresh_log (view_name, run_at, status, error_message)
        VALUES ('customer_kpi_summary', v_start, 'sla_breach',
                'Refresh exceeded 45-minute window: ' || v_duration_sec || 's');
    END IF;

EXCEPTION WHEN OTHERS THEN
    INSERT INTO audit.mv_refresh_log (
        view_name, run_at, duration_seconds, status, error_message
    ) VALUES (
        'customer_kpi_summary', v_start,
        EXTRACT(EPOCH FROM NOW() - v_start),
        'failed', SQLERRM
    );
    RAISE;
END;
$$;
```

### 4. Control-M Scheduling

**Job chain configuration (daily at 18:30):**

```
[UPSTREAM JOBS]
  data_ingestion_complete    ─┐
  risk_scores_loaded         ─┤──► [kpi_mv_refresh]  ──► [dashboard_cache_warm]
  credit_exposure_updated    ─┘           │
                                     On failure:
                                       ► alert_on_call
                                       ► hold downstream jobs
```

**Key scheduling decisions:**
- **18:30 trigger** — after all source data pipelines complete, before end-of-day reporting
- **Dependency chain** — refresh blocked until upstream jobs confirm success
- **Downstream hold** — dashboard cache warming waits for successful refresh
- **Max runtime** — job killed and alerted if exceeding 45-minute SLA

### 5. Dashboard Query — After Optimization

```sql
-- Sub-second report query against materialized view
SELECT
    segment,
    risk_tier,
    COUNT(*)                        AS customer_count,
    SUM(total_spend)                AS portfolio_spend,
    AVG(spend_30d)                  AS avg_recent_spend,
    SUM(total_exposure)             AS total_exposure,
    COUNT(*) FILTER (
        WHERE current_risk_score >= 70
    )                               AS high_risk_customers,
    MAX(refreshed_at)               AS data_as_of
FROM analytics.customer_kpi_summary
WHERE last_activity_date >= CURRENT_DATE - 90
GROUP BY segment, risk_tier
ORDER BY total_exposure DESC;
-- Execution time: ~80ms (vs 10+ minutes on source tables)
```

---

## 📈 Performance Results

| Metric | Before | After | Improvement |
|--------|--------|-------|-------------|
| **Dashboard query time** | 10+ minutes | <1 second | >95% |
| **Production DB load** | High (daytime) | None (daytime) | Eliminated |
| **Report availability** | Inconsistent | Predictable daily | 100% consistent |
| **Refresh reliability** | Manual / ad-hoc | 99.5% automated | Fully automated |
| **Refresh duration** | Uncontrolled | Within 45-min SLA | Monitored & bounded |

---

## 💼 Business Value

- **User experience** — analysts and executives access reports instantly during business hours
- **System stability** — production database no longer contends with heavy analytical queries
- **Operational efficiency** — manual refresh tasks eliminated; on-call only engaged on genuine failures
- **Scalability** — denormalized structure and partitioned indexes support 10x data volume without redesign

---

## 🎯 Key Learnings

1. **Move aggregation to refresh time, not query time** — pre-calculating rolling windows at 18:30 means dashboards never pay that cost

2. **CONCURRENTLY is non-negotiable in production** — blocking refresh locks the view; concurrent refresh requires a unique index but keeps dashboards live during refresh

3. **Index on access patterns, not schema** — composite indexes built around actual dashboard filter combinations, not normalized FK columns

4. **Control-M dependency chains prevent dirty reads** — upstream job confirmation gates refresh; avoids partial data in dashboard

---

## 📚 Technologies Used

- **PostgreSQL** — materialized views, stored procedures, composite indexing, CONCURRENTLY refresh
- **Control-M** — job scheduling, dependency management, SLA monitoring, failure alerting
- **SQL** — query optimization, window functions, FILTER aggregates, execution plan analysis

---

[← Back to Portfolio](../README.md)
