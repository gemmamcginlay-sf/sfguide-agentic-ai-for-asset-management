# ORBIT Transformation Plan

This document guides the phased transformation of SAM → Barclays ORBIT. Execute each phase sequentially. **Verify each phase works before proceeding to the next.**

---

## Pre-Requisites (Completed)

- [x] Repo deployed from `gemmamcginlay-sf/sfguide-agentic-ai-for-asset-management` (v2 branch)
- [x] `SNOWFLAKE_PUBLIC_DATA_PAID` already installed in account
- [x] `setup.sql` executed successfully (with manual edit to point to Paid listing)
- [x] `workspace_main.py` completed the full SAM build successfully
- [x] All 8 SAM agents respond in CoWork

---

## Phase 1: Rebrand SAM → ORBIT

**Goal:** Replace all SAM naming with ORBIT. No structural changes. Everything still works identically.

### 1.1 Configuration (`python/config.py`)

Replace the following — these are the ONLY source-of-truth values; everything else derives from them:

| Old | New |
|-----|-----|
| `'name': 'SAM_DEMO'` | `'name': 'ORBIT_DEMO'` |
| `'SAM_DEMO_EXECUTION_WH'` | `'ORBIT_DEMO_EXECUTION_WH'` |
| `'SAM_DEMO_CORTEX_WH'` | `'ORBIT_DEMO_CORTEX_WH'` |
| `'portfolio_code_prefix': 'SAM'` | `'portfolio_code_prefix': 'ORBIT'` |
| All portfolio names: `'SAM Technology...'` | `'ORBIT Technology...'` |
| Agent names: `'AM_portfolio_management_copilot'` etc. | `'ORBIT_portfolio_management_copilot'` etc. |
| All `'SAM_*_VIEW'` references | `'ORBIT_*_VIEW'` |
| All `'SAM_*'` search service references | `'ORBIT_*'` |
| `Simulated Asset Management (SAM)` | `Barclays ORBIT` |

### 1.2 Data Source (`python/config.py`)

```python
# Change these in REAL_DATA_SOURCES:
'database': 'SNOWFLAKE_PUBLIC_DATA_PAID',
'schema': 'PUBLIC_DATA',
```

### 1.3 Setup SQL (`scripts/setup.sql`)

- Replace all `SAM_DEMO` → `ORBIT_DEMO`
- Replace all `SAM_DEMO_ROLE` → `ORBIT_DEMO_ROLE`
- Replace warehouse names
- Remove/comment `SYSTEM$REQUEST_LISTING_AND_WAIT('GZTSZ290BV255')` (Paid listing already installed)
- Remove `CREATE DATABASE IF NOT EXISTS SNOWFLAKE_PUBLIC_DATA_FREE FROM LISTING`
- Change `SNOWFLAKE_PUBLIC_DATA_FREE` → `SNOWFLAKE_PUBLIC_DATA_PAID`

### 1.4 Agent Files (`python/ai/agents/*.py`)

- Replace `AM_` prefix with `ORBIT_` in all agent object names
- Replace `Simulated Asset Management`/`SAM` firm references with `Barclays ORBIT`/`ORBIT`
- Replace all `SAM ` portfolio name prefixes with `ORBIT `

### 1.5 Semantic View YAMLs (`python/ai/semantic_view_definitions/*.yaml`)

- Rename files: `SAM_*.yaml` → `ORBIT_*.yaml`
- Inside each YAML: update `name:` field and any database references
- Update portfolio name references

### 1.6 Content Library, Skills, Reference Data

- Bulk replace `SAM` → `ORBIT` in all `content_library/`, `data/skills/`, `data/reference_data/` files
- **Exception:** Keep `SAM_BILLIONS` (this is "Serviceable Addressable Market" — a finance term, not our brand)

### 1.7 Verification

```sql
-- After rebuild, verify:
SHOW DATABASES LIKE 'ORBIT_DEMO';
SHOW CORTEX SEARCH SERVICES IN ORBIT_DEMO.AI;
SHOW AGENTS IN ORBIT_DEMO.AI;
SELECT * FROM TABLE(INFORMATION_SCHEMA.SEMANTIC_VIEW_USAGE()) WHERE SEMANTIC_VIEW_SCHEMA = 'AI';
```

Test each agent in CoWork responds correctly.

---

## Phase 2: Strip to Real-Data Middle Ground

**Goal:** Remove agents/components that depend on entirely synthetic data. Keep components where data is real or defensibly derived from real data.

### 2.1 Agents to KEEP (5)

| Agent | Justification |
|---|---|
| **Portfolio Management** | Model portfolios of real stocks at real prices. Attribution is maths on real returns. Monte Carlo uses real covariance. |
| **Research** | SEC filings, transcripts, financials — all real from Marketplace |
| **Risk & Compliance** | Concentration risk on real positions, real SEC filings for regulatory |
| **Executive** | Real macro data, institutional holdings, market intelligence |
| **Client Advisory** (simplified) | Can work with real portfolio data + real market context. Remove dependency on fake client personas. |

### 2.2 Agents to REMOVE (3)

| Agent | Reason |
|---|---|
| **Operations Copilot** | 100% synthetic settlements, NAV, reconciliation |
| **Private Equity Copilot** | 100% synthetic deal pipeline |
| **Private Credit Copilot** | 100% synthetic borrowers/covenants |

### 2.3 What to Remove

- Delete agent files: `middle_office_copilot.py`, `pe_copilot.py`, `private_credit.py`
- Remove from `python/ai/agents/__init__.py` AGENT_CREATORS dict
- Remove scenarios from config: `operations`, `private_equity`, `private_credit`
- Remove ML scenarios: `market_regime_ml`, `factor_workflow_ml`, `credit_risk_ml`
- Delete semantic views: `ORBIT_MIDDLE_OFFICE_VIEW`, `ORBIT_PE_*_VIEW`, `ORBIT_CREDIT_*_VIEW`
- Delete search services for fake docs: `ORBIT_EXTERNAL_DOCS`, `ORBIT_INTERNAL_DOCS`, `ORBIT_REGULATORY_DOCS`, all PE/Credit search services
- Delete `content_library/` entirely (all template-generated fake PDFs)
- Delete `python/data/unstructured.py`, `python/core/hydration_engine.py`, `python/core/pdf_exporter.py`
- Delete `python/data/pipelines.py` (document pipeline infrastructure)
- Remove ~25 skills that depend on removed agents/synthetic data

### 2.4 What to KEEP

- All real Marketplace data loading (`market_data.py`)
- Model portfolio generation from real prices (`structured.py` — positions, transactions)
- Returns, covariance, factor exposures (derived from real data)
- Attribution (Brinson on real benchmark ETF returns)
- Monte Carlo, Backtest, Counterfactual tools
- Real search services: `ORBIT_REAL_SEC_FILINGS`, `ORBIT_COMPANY_EVENTS`
- Semantic views: `ORBIT_PORTFOLIO_VIEW`, `ORBIT_RESEARCH_VIEW`, `ORBIT_MARKET_VIEW`, `ORBIT_PORTFOLIO_MODELLING_VIEW`, `ORBIT_ATTRIBUTION_VIEW`
- ~10 real-data skills (earnings-intelligence, equity-research, competitive-intelligence, etc.)
- Transcript processing (`transcripts.py`)

### 2.5 Verification

Rebuild and verify all 5 remaining agents work in CoWork. Key test queries:
- Portfolio: "Show me top holdings in ORBIT Technology & Infrastructure"
- Research: "Deep dive on NVIDIA's last earnings call"
- Risk: "What's our concentration in technology stocks?"
- Executive: "Show me Treasury yield curve changes this month"

---

## Phase 3: Convert to SQL (For Barclays Deployability)

**Goal:** Replace the Python workspace build with SQL scripts that can be run from SQL worksheets. This removes the dependency on compute pools and Python runtimes.

### 3.1 Create SQL Scripts

Create a `scripts/` directory with numbered SQL files:

```
scripts/
├── 01_setup_infrastructure.sql    (from current setup.sql)
├── 02_dimension_tables.sql        (DIM_ISSUER, DIM_SECURITY, DIM_PORTFOLIO, DIM_BENCHMARK)
├── 03_market_data.sql             (all FACT_ tables from Marketplace — CTAS statements)
├── 04_derived_analytics.sql       (V_SECURITY_RETURNS, FACT_COVARIANCE_MATRIX, FACT_FACTOR_EXPOSURES)
├── 05_model_portfolios.sql        (positions, transactions from real prices)
├── 06_attribution.sql             (benchmark returns, Brinson tables)
├── 07_transcript_corpus.sql       (load + chunk real transcripts)
├── 08_search_services.sql         (CREATE CORTEX SEARCH SERVICE DDL)
├── 09_semantic_views.sql          (CREATE SEMANTIC VIEW from YAML content)
├── 10_tools.sql                   (UDFs/SPs for Monte Carlo, backtest, attribution)
├── 11_agents.sql                  (CREATE AGENT DDL + register with Intelligence)
├── 12_daily_refresh.sql           (Tasks for scheduled data refresh)
└── 99_teardown.sql                (DROP everything)
```

### 3.2 Dynamic Tables for Auto-Refresh

Where appropriate, use Dynamic Tables instead of static CTAS:

```sql
-- Example: FACT_STOCK_PRICES refreshes daily from Marketplace
CREATE OR REPLACE DYNAMIC TABLE ORBIT_DEMO.MARKET_DATA.FACT_STOCK_PRICES
    TARGET_LAG = '1 day'
    WAREHOUSE = ORBIT_DEMO_EXECUTION_WH
AS
SELECT TICKER, DATE, VARIABLE, VALUE
FROM SNOWFLAKE_PUBLIC_DATA_PAID.PUBLIC_DATA.STOCK_PRICE_TIMESERIES
WHERE TICKER IN (SELECT PRIMARY_TICKER FROM ORBIT_DEMO.CURATED.DIM_ISSUER)
  AND VARIABLE IN ('post-market_close', 'pre-market_open', 'all-day_high', 'all-day_low', 'all-day_volume');
```

### 3.3 Tasks for Daily Refresh

```sql
-- Daily refresh task for data that can't be Dynamic Tables
CREATE OR REPLACE TASK ORBIT_DEMO.AI.DAILY_REFRESH
    WAREHOUSE = ORBIT_DEMO_EXECUTION_WH
    SCHEDULE = 'USING CRON 0 6 * * * UTC'  -- 6 AM UTC daily
AS
CALL ORBIT_DEMO.AI.REFRESH_MARKET_DATA();
```

Tables that should refresh daily:
- `FACT_STOCK_PRICES` — new daily prices
- `FACT_SEC_FINANCIALS` — new filings as they come in
- `FACT_INSIDER_TRANSACTIONS` — new Form 4s
- `FACT_INSTITUTIONAL_HOLDINGS` — quarterly (13F lag)
- `FACT_TREASURY_YIELDS` — daily
- `FACT_ECONOMIC_INDICATORS` — variable (FRED updates)
- Cortex Search services already auto-refresh (5-minute target lag)

### 3.4 Verification

Run all 12 SQL scripts in order. Verify same result as Python build.

---

## Phase 4: Future Enhancements

| Enhancement | What It Enables | When |
|---|---|---|
| Bloomberg Second Measure | Consumer spending signals | When commercial agreement in place |
| Real portfolio feed | Actual Barclays holdings | When IB data team provides export |
| ORBIT branding/logo | Custom Streamlit hub | When Sahana provides brand assets |
| ESG ratings (MSCI) | Real ESG scores vs synthetic | When Marketplace listing approved |
| Additional agents | Ops/PE/Credit with real data | When real internal data available |

---

## Execution Notes

- Execute ONE phase at a time. Verify before proceeding.
- If any step fails, debug before continuing — don't accumulate broken state.
- The Python build (workspace_main.py) is the reference implementation — use it to understand what each SQL script should produce.
- After Phase 3, the Python workspace is no longer needed for deployment (but useful for development/testing).
- Commit after each successful phase.
