# ORBIT Transformation — CoCo UI Prompts (Sequential)

Use these prompts in order, in the Snowsight CoCo UI, after the base SAM demo is running. Each prompt is self-contained. Execute one, verify it works, then move to the next.

---

## Phase 1: Rebrand SAM → ORBIT

### Prompt 1.1 — Rename Database & Roles

```
Rename the database from SAM_DEMO to ORBIT_DEMO. Update the role from SAM_DEMO_ROLE to ORBIT_DEMO_ROLE. Update all warehouse names from SAM_DEMO_* to ORBIT_DEMO_*. Do this by running ALTER statements, then update scripts/setup.sql and python/config.py to reflect the new names. Update any references throughout the codebase. The role should still own the database and warehouses.
```

### Prompt 1.2 — Rename Semantic Views

```
Rename all semantic views in ORBIT_DEMO.AI schema from SAM_* to ORBIT_*. For each, run:
ALTER SEMANTIC VIEW ORBIT_DEMO.AI.SAM_<name> RENAME TO ORBIT_DEMO.AI.ORBIT_<name>;
Then update the corresponding YAML files in python/ai/semantic_view_definitions/ to use ORBIT_ prefixes in their names.
```

### Prompt 1.3 — Rename Agents

```
Rename all Cortex Agents from SAM_* to ORBIT_*. For each agent in ORBIT_DEMO.AI schema, run:
ALTER CORTEX AGENT ORBIT_DEMO.AI.SAM_<name> RENAME TO ORBIT_DEMO.AI.ORBIT_<name>;
Update python/ai/agents/__init__.py and any other references. The agents are: PORTFOLIO_MANAGEMENT, RESEARCH, RISK_COMPLIANCE, EXECUTIVE_SUMMARY, CLIENT_ADVISORY.
```

### Prompt 1.4 — Rename Search Services

```
Rename Cortex Search Services from SAM_* to ORBIT_*. Note: Search Services cannot be renamed with ALTER — you need to DROP and recreate them with the new name. For each search service in ORBIT_DEMO.AI schema, capture its definition first (DESCRIBE), then DROP and CREATE with the ORBIT_ prefix. Services: SAM_SEC_FILINGS_SEARCH, SAM_EARNINGS_TRANSCRIPT_SEARCH.
```

---

## Phase 2: Verify Real Data Sources

### Prompt 2.1 — Confirm Paid Data

```
Check that SNOWFLAKE_PUBLIC_DATA_PAID.PUBLIC_DATA is accessible and has recent data. Run:
SELECT MAX(DATE) AS latest_date, COUNT(*) AS total_rows FROM SNOWFLAKE_PUBLIC_DATA_PAID.PUBLIC_DATA.FACT_STOCK_PRICES;
SELECT MAX(DATE) AS latest_date FROM SNOWFLAKE_PUBLIC_DATA_PAID.PUBLIC_DATA.FACT_TREASURY_YIELDS;
SELECT MAX(DATE) AS latest_date FROM SNOWFLAKE_PUBLIC_DATA_PAID.PUBLIC_DATA.FACT_ECONOMIC_INDICATORS;
Report the latest dates for each to confirm near-real-time freshness.
```

### Prompt 2.2 — Confirm Real Components Map

```
For the current state of the ORBIT_DEMO database, classify every table/view in schemas CURATED, MARKET_DATA, and AI into: (A) Real Data — sourced from Snowflake Public Data Paid, (B) Derived from Real Data — calculated from real data (like benchmark returns from ETF prices), (C) Synthetic/Model Data — using model portfolios or generated data. List the category for each table with a brief note on the source. This helps me decide what to keep.
```

---

## Phase 3: SQL-First Conversion

### Prompt 3.1 — Generate DDL from Python

```
Look at python/data/structured.py and python/data/market_data.py. These contain Python functions that create views/tables in ORBIT_DEMO. For each function, generate the equivalent SQL CREATE VIEW or CREATE TABLE AS SELECT statement and put it in a new file scripts/curated_views.sql. This should be pure SQL that can run directly without Python. Keep the same logic — just translate the Snowpark DataFrame operations to SQL.
```

### Prompt 3.2 — Generate AI Object DDL

```
Look at python/ai/builder.py and the YAML files in python/ai/semantic_view_definitions/. Generate SQL DDL for all AI objects:
1. CREATE CORTEX SEARCH SERVICE statements (from the Python search service builder logic)
2. CREATE SEMANTIC VIEW statements (from the YAML definitions)
3. CREATE CORTEX AGENT statements (from the agent definitions)
Put all of this in scripts/ai_objects.sql. This should be a standalone SQL script that creates all AI layer objects assuming the data layer already exists.
```

### Prompt 3.3 — Create Master Deployment Script

```
Create a single scripts/deploy.sql that orchestrates the full ORBIT deployment in order:
1. Source scripts/setup.sql (infra: roles, database, schemas, warehouses, grants)
2. Source scripts/curated_views.sql (data layer: views on top of Paid data)
3. Source scripts/ai_objects.sql (AI layer: search services, semantic views, agents)
Each step should be idempotent (use CREATE OR REPLACE where possible). Add comments between sections. This becomes the single entry point for deploying ORBIT from scratch.
```

---

## Phase 4: Daily Refresh

### Prompt 4.1 — Dynamic Tables

```
For the derived data tables in ORBIT_DEMO (like FACT_BENCHMARK_RETURNS which is calculated from ETF prices), convert them to Dynamic Tables with a TARGET_LAG of '1 day'. This means they auto-refresh daily when the underlying Paid data updates. Generate the CREATE DYNAMIC TABLE statements and add them to scripts/curated_views.sql replacing the static CREATE TABLE AS SELECT versions.
```

### Prompt 4.2 — Scheduled Task for Search Service Refresh

```
Cortex Search Services need periodic refresh as new SEC filings and earnings transcripts arrive. Create a Snowflake Task that runs daily at 06:00 UTC to:
1. Refresh the transcript staging table (if new data available)
2. Refresh the SEC filings staging table
The search services themselves auto-refresh on their source tables, so we just need to ensure the staging tables are current. Put this in scripts/daily_refresh.sql.
```

---

## Phase 5: Streamlit Portal

### Prompt 5.1 — Build the Portal

```
See the full specification in docs/ORBIT_STREAMLIT_PORTAL_PROMPT.md. Build the Streamlit application as described there. Start with the landing page (Page 1: Home/Dashboard) and get it working, then add pages one at a time. Use the ORBIT branding (colours, logo from assets/logos/). The app should connect to ORBIT_DEMO using get_active_session().
```

---

## Verification Commands

Run these after each phase to verify things work:

```sql
-- After Phase 1
SHOW DATABASES LIKE 'ORBIT%';
SHOW SCHEMAS IN DATABASE ORBIT_DEMO;
SHOW CORTEX AGENTS IN SCHEMA ORBIT_DEMO.AI;
SHOW CORTEX SEARCH SERVICES IN SCHEMA ORBIT_DEMO.AI;

-- After Phase 2
SELECT COUNT(*) FROM ORBIT_DEMO.MARKET_DATA.FACT_STOCK_PRICES WHERE DATE = CURRENT_DATE();

-- After Phase 3
-- Run scripts/deploy.sql end-to-end in a fresh environment

-- After Phase 4
SHOW DYNAMIC TABLES IN SCHEMA ORBIT_DEMO.CURATED;
SHOW TASKS IN SCHEMA ORBIT_DEMO.CURATED;

-- After Phase 5
SHOW STREAMLITS IN SCHEMA ORBIT_DEMO.PUBLIC;
```
