# Production-Ready PostgreSQL Development Environment

**Company:** FinTech/Banking
**Role:** BI / Data Developer
**Tech Stack:** Docker, PostgreSQL, Python, Git, Bash

---

## 🎯 Project Goals

Built a comprehensive local development environment mirroring production database architecture — enabling safe development, testing, and skill advancement without impacting live systems.

---

## 🏗️ Solution Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    Docker Container                          │
│                                                             │
│  ┌──────────────────────────────────────────────────────┐  │
│  │              PostgreSQL Database                      │  │
│  │  • Multi-schema design (finance, analytics, audit)   │  │
│  │  • Tables with FK constraints                        │  │
│  │  • Materialized views                                │  │
│  │  • Custom functions & stored procedures              │  │
│  │  • Role-based access control                         │  │
│  └───────────────────┬──────────────────────────────────┘  │
└──────────────────────┼──────────────────────────────────────┘
                       │
          ┌────────────┴────────────┐
          ▼                        ▼
┌──────────────────┐    ┌──────────────────────┐
│  Python Scripts  │    │    Bash Automation   │
│  • Query runner  │    │  • Setup scripts     │
│  • Data testing  │    │  • Backup/recovery   │
│  • Validation    │    │  • Env deployment    │
└──────────────────┘    └──────────────────────┘
          │
          ▼
┌──────────────────┐
│   Git / GitHub   │
│  • Schema DDL    │
│  • Test suites   │
│  • Config files  │
└──────────────────┘
```

---

## ⚙️ Technical Implementation

### 1. Database Architecture

**Multi-Schema Design:**

```sql
-- Schema structure simulating enterprise architecture
CREATE SCHEMA finance;
CREATE SCHEMA analytics;
CREATE SCHEMA audit;
CREATE SCHEMA staging;

-- Core tables with proper constraints
CREATE TABLE finance.customers (
    customer_id     SERIAL PRIMARY KEY,
    external_id     VARCHAR(50) UNIQUE NOT NULL,
    created_at      TIMESTAMP DEFAULT NOW(),
    status          VARCHAR(20) CHECK (status IN ('active', 'inactive', 'suspended'))
);

CREATE TABLE finance.transactions (
    transaction_id  SERIAL PRIMARY KEY,
    customer_id     INT NOT NULL REFERENCES finance.customers(customer_id),
    amount          NUMERIC(15, 2) NOT NULL,
    transaction_date DATE NOT NULL,
    category        VARCHAR(50),
    created_at      TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_transactions_customer ON finance.transactions(customer_id);
CREATE INDEX idx_transactions_date ON finance.transactions(transaction_date);
```

**Custom Functions & Stored Procedures:**

```sql
-- Reusable business logic in database layer
CREATE OR REPLACE FUNCTION finance.get_customer_monthly_summary(
    p_customer_id INT,
    p_months INT DEFAULT 3
)
RETURNS TABLE (
    month           DATE,
    total_spend     NUMERIC,
    transaction_count INT,
    avg_transaction NUMERIC
) AS $$
BEGIN
    RETURN QUERY
    SELECT
        DATE_TRUNC('month', transaction_date)::DATE,
        SUM(amount),
        COUNT(*)::INT,
        AVG(amount)
    FROM finance.transactions
    WHERE customer_id = p_customer_id
      AND transaction_date >= CURRENT_DATE - (p_months || ' months')::INTERVAL
    GROUP BY DATE_TRUNC('month', transaction_date)
    ORDER BY 1;
END;
$$ LANGUAGE plpgsql;

-- Stored procedure with error handling
CREATE OR REPLACE PROCEDURE finance.refresh_analytics_layer()
LANGUAGE plpgsql AS $$
DECLARE
    v_start TIMESTAMP := NOW();
BEGIN
    REFRESH MATERIALIZED VIEW CONCURRENTLY analytics.customer_summary;
    REFRESH MATERIALIZED VIEW CONCURRENTLY analytics.monthly_kpis;

    INSERT INTO audit.refresh_log (run_at, duration_seconds, status)
    VALUES (v_start, EXTRACT(EPOCH FROM NOW() - v_start), 'success');

EXCEPTION WHEN OTHERS THEN
    INSERT INTO audit.refresh_log (run_at, status, error_message)
    VALUES (v_start, 'failed', SQLERRM);
    RAISE;
END;
$$;
```

**Role-Based Access Control:**

```sql
-- Roles simulating production access tiers
CREATE ROLE readonly_role;
CREATE ROLE analyst_role;
CREATE ROLE developer_role;

GRANT USAGE ON SCHEMA finance, analytics TO readonly_role;
GRANT SELECT ON ALL TABLES IN SCHEMA finance TO readonly_role;
GRANT SELECT ON ALL TABLES IN SCHEMA analytics TO readonly_role;

GRANT readonly_role TO analyst_role;
GRANT INSERT, UPDATE ON finance.transactions TO analyst_role;

GRANT analyst_role TO developer_role;
GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA staging TO developer_role;
```

### 2. Materialized Views

```sql
-- Pre-aggregated customer summary for fast dashboard queries
CREATE MATERIALIZED VIEW analytics.customer_summary AS
SELECT
    c.customer_id,
    c.external_id,
    c.status,
    COUNT(t.transaction_id)                     AS total_transactions,
    SUM(t.amount)                               AS total_spend,
    AVG(t.amount)                               AS avg_transaction,
    MAX(t.transaction_date)                     AS last_transaction_date,
    MIN(t.transaction_date)                     AS first_transaction_date,
    SUM(t.amount) FILTER (
        WHERE t.transaction_date >= CURRENT_DATE - 30
    )                                           AS spend_last_30d
FROM finance.customers c
LEFT JOIN finance.transactions t USING (customer_id)
GROUP BY c.customer_id, c.external_id, c.status;

CREATE UNIQUE INDEX ON analytics.customer_summary(customer_id);
```

### 3. Python Data Access

```python
# db_client.py
import os
import psycopg2
from psycopg2 import pool
from contextlib import contextmanager
import pandas as pd


class DatabaseClient:
    def __init__(self):
        self._pool = psycopg2.pool.ThreadedConnectionPool(
            minconn=2,
            maxconn=10,
            host=os.environ['DB_HOST'],
            port=os.environ.get('DB_PORT', 5432),
            dbname=os.environ['DB_NAME'],
            user=os.environ['DB_USER'],
            password=os.environ['DB_PASSWORD'],
        )

    @contextmanager
    def connection(self):
        conn = self._pool.getconn()
        try:
            yield conn
            conn.commit()
        except Exception:
            conn.rollback()
            raise
        finally:
            self._pool.putconn(conn)

    def query(self, sql: str, params=None) -> pd.DataFrame:
        with self.connection() as conn:
            return pd.read_sql(sql, conn, params=params)

    def execute(self, sql: str, params=None) -> None:
        with self.connection() as conn:
            with conn.cursor() as cur:
                cur.execute(sql, params)
```

### 4. Automated Validation & Testing

```python
# test_database.py
import pytest
from db_client import DatabaseClient

db = DatabaseClient()


def test_fk_integrity():
    result = db.query("""
        SELECT COUNT(*) AS orphans
        FROM finance.transactions t
        LEFT JOIN finance.customers c USING (customer_id)
        WHERE c.customer_id IS NULL
    """)
    assert result['orphans'].iloc[0] == 0, "FK violation: orphaned transactions found"


def test_customer_summary_completeness():
    result = db.query("""
        SELECT
            (SELECT COUNT(*) FROM finance.customers)           AS source_count,
            (SELECT COUNT(*) FROM analytics.customer_summary) AS view_count
    """)
    assert result['source_count'].iloc[0] == result['view_count'].iloc[0]


def test_monthly_summary_function():
    result = db.query(
        "SELECT * FROM finance.get_customer_monthly_summary(%s, %s)",
        params=(1, 3)
    )
    assert not result.empty
    assert 'total_spend' in result.columns
```

### 5. Bash Automation

```bash
#!/bin/bash
# setup.sh — single-command environment deployment

set -euo pipefail

echo "Starting PostgreSQL dev environment..."

# Start container
docker-compose up -d postgres
sleep 3

# Run schema migrations in order
for f in sql/01_schemas.sql sql/02_tables.sql sql/03_functions.sql \
          sql/04_views.sql sql/05_roles.sql sql/06_seed_data.sql; do
    echo "Applying $f..."
    docker exec -i dev_postgres psql -U "$DB_USER" -d "$DB_NAME" < "$f"
done

# Verify deployment
python3 -m pytest tests/test_database.py -v

echo "Environment ready."
```

---

## 📈 Outcomes

| Area | Result |
|------|--------|
| **Development Safety** | Zero risk to production — all experimentation isolated |
| **Setup Time** | Full environment deployed in under 2 minutes |
| **Test Coverage** | Automated validation for FK integrity, view completeness, function correctness |
| **Reproducibility** | Identical environment across machines via Docker |

---

## 🎯 Key Learnings

1. **Database logic vs application logic trade-offs** — Stored procedures are powerful but harder to test and version; Python gives more flexibility for complex orchestration

2. **Container consistency** — Docker eliminates "works on my machine" issues; environment variables keep credentials out of code

3. **Test-first data development** — Writing validation tests before data logic catches schema issues early

4. **RBAC from the start** — Designing role separation upfront avoids painful retrofitting later

---

## 📚 Technologies Used

- **PostgreSQL** — schemas, tables, functions, stored procedures, materialized views, RBAC
- **Docker** — containerized environment, `docker-compose` orchestration
- **Python** — `psycopg2`, connection pooling, `pytest` test suite, `pandas`
- **Git/GitHub** — version-controlled DDL, schema migrations
- **Bash** — automated setup, backup/recovery scripts

---

[← Back to Portfolio](../README.md)
