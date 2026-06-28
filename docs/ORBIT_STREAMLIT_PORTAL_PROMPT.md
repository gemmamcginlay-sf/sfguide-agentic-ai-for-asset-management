# ORBIT Streamlit Portal — Build Instructions for CoCo UI

This document provides everything needed to build the ORBIT Streamlit portal application. Give this to CoCo in the Snowsight workspace UI after the SAM demo is deployed and the ORBIT transformation (Phase 1 + Phase 2 from `docs/ORBIT_TRANSFORMATION_PLAN.md`) is complete.

---

## Prompt to Give CoCo UI

Copy everything below the line and paste it as your prompt to CoCo in the Snowsight workspace:

---

## START OF PROMPT

I need you to build a Streamlit application that serves as the **ORBIT Portal** — a branded hub for the Barclays ORBIT investment intelligence platform.

### What ORBIT Is

ORBIT = "Omnicient Reasoning Barclays Intelligence Tool". It's a multi-agent investment intelligence platform powered by Snowflake Cortex. The platform has:
- 5 Cortex Agents (Portfolio Management, Research, Risk & Compliance, Executive, Client Advisory)
- Real-time market data from Snowflake Public Data (Paid) — 14,000+ securities
- Cortex Search over real SEC filings and earnings transcripts
- Semantic views for text-to-SQL queries on financial data

### Brand Guidelines

- **Primary colour:** `#1B6B93` (deep teal-blue)
- **Secondary colour:** `#2E8BC0` (medium teal)
- **Accent:** `#5DADE2` (light blue glow)
- **Dark background:** `#1B2838` (navy)
- **Text on dark:** `#FFFFFF`
- **Text on light:** `#1A3A5C` (navy)
- **Font:** Clean sans-serif (use Streamlit default)
- **Logo files:** Located in `assets/logos/` directory
  - `orbit_logo_dark_square.png` — square icon for dark backgrounds
  - `orbit_logo_light_square.png` — square icon for light backgrounds
  - `orbit_logo_dark_horizontal.png` — horizontal with "Omnicient Reasoning Barclays Intelligence Tool" tagline, dark bg
  - `orbit_logo_light_horizontal.png` — horizontal with tagline, light bg

### Application Structure

Build a **multi-page Streamlit app** with the following pages:

#### Page 1: Home / Dashboard (Landing Page)
- ORBIT logo (horizontal variant) centred at top
- Tagline: "Omnicient Reasoning Barclays Intelligence Tool"
- 4 metric cards showing live data:
  - Total securities tracked (count from DIM_ISSUER)
  - Latest market date (max date from FACT_STOCK_PRICES)
  - SEC filings loaded (count from FACT_SEC_FINANCIALS)
  - Earnings transcripts available (count from transcript corpus table)
- Grid of tiles linking to other pages (Research, Portfolio, Market, Risk)
- Clean, professional layout with ORBIT branding

#### Page 2: Market Intelligence
- Real-time market overview using data from ORBIT_DEMO.MARKET_DATA
- Treasury yield curve chart (from FACT_TREASURY_YIELDS — latest date)
- Key economic indicators dashboard (FACT_ECONOMIC_INDICATORS — GDP, CPI, unemployment)
- FX rates table (FACT_FX_RATES — latest)
- Central bank policy rates comparison (FACT_POLICY_RATES)

#### Page 3: Research Hub
- Company search (text input → query DIM_ISSUER)
- When company selected, show:
  - Stock price chart (FACT_STOCK_PRICES — last 12 months)
  - Revenue by segment (FACT_SEC_SEGMENTS — latest quarter)
  - Key financials table (FACT_SEC_FINANCIALS — Revenue, Net Income, EPS, last 4 quarters)
  - Insider trading activity (FACT_INSIDER_TRANSACTIONS — last 90 days)
  - Top institutional holders (FACT_INSTITUTIONAL_HOLDINGS — latest filing)

#### Page 4: Portfolio Overview
- Dropdown to select portfolio (from DIM_PORTFOLIO)
- Holdings table with current weights
- Sector allocation pie chart
- Top 10 holdings bar chart
- Portfolio value over time (if positions data available)

#### Page 5: AI Agents
- Cards for each ORBIT agent with:
  - Agent name and description
  - `st.link_button` that deep-links directly into CoWork with the agent pre-selected
  - Example questions you can ask each agent (shown as clickable suggestions)
- This page serves as a guide to the AI capabilities AND the launch pad into conversational AI
- Use this pattern for the CoWork deep-links:

```python
import streamlit as st
from snowflake.snowpark.context import get_active_session

session = get_active_session()
account_url = f"https://{session.get_current_account()}.snowflakecomputing.com"

agents = {
    "Portfolio Management": {
        "fqn": "ORBIT_DEMO.AI.ORBIT_PORTFOLIO_MANAGEMENT",
        "description": "Ask about portfolio holdings, performance, allocation, and rebalancing.",
        "examples": [
            "What is the current sector allocation of Portfolio Alpha?",
            "Which holdings contributed most to returns this quarter?",
            "Show me the top 10 positions by weight",
        ]
    },
    "Research": {
        "fqn": "ORBIT_DEMO.AI.ORBIT_RESEARCH",
        "description": "Deep-dive on companies: financials, SEC filings, earnings, insider activity.",
        "examples": [
            "Summarise Apple's latest quarterly earnings",
            "What did Microsoft's CEO say about AI in the last earnings call?",
            "Show me insider transactions for Tesla in the last 90 days",
        ]
    },
    "Risk & Compliance": {
        "fqn": "ORBIT_DEMO.AI.ORBIT_RISK_COMPLIANCE",
        "description": "Analyse risk exposures, concentration, and compliance constraints.",
        "examples": [
            "What is our tech sector concentration risk?",
            "Are any holdings breaching the 5% single-name limit?",
            "Show factor exposures for Portfolio Alpha",
        ]
    },
    "Executive Summary": {
        "fqn": "ORBIT_DEMO.AI.ORBIT_EXECUTIVE_SUMMARY",
        "description": "High-level portfolio and market summaries for senior stakeholders.",
        "examples": [
            "Give me a one-page summary of portfolio performance this month",
            "What are the key market risks heading into next week?",
        ]
    },
    "Client Advisory": {
        "fqn": "ORBIT_DEMO.AI.ORBIT_CLIENT_ADVISORY",
        "description": "Client-facing insights and investment recommendations.",
        "examples": [
            "Draft a client note on our current market outlook",
            "What themes should we be discussing with growth-oriented clients?",
        ]
    },
}

for name, agent in agents.items():
    with st.container(border=True):
        st.subheader(name)
        st.write(agent["description"])
        cowork_url = f"{account_url}/intelligence/cowork?agent={agent['fqn']}"
        st.link_button(f"Chat with {name} Agent →", cowork_url)
        st.caption("Example questions:")
        for ex in agent["examples"]:
            st.markdown(f"- _{ex}_")
```

- The `st.link_button` opens CoWork in a new tab with the specific agent pre-loaded — the user lands directly in a conversation

### Technical Requirements

- Use `snowflake.snowpark.context.get_active_session()` for the Snowflake connection
- Database: `ORBIT_DEMO` (or read from environment)
- All queries should use the ORBIT_DEMO schemas: CURATED, MARKET_DATA, AI
- Use `st.set_page_config(page_title="ORBIT", page_icon="assets/logos/orbit_logo_dark_square.png", layout="wide")`
- Apply custom CSS for ORBIT colours via `st.markdown` with `unsafe_allow_html=True`
- The app should be a single `streamlit_app.py` in the workspace root (or in a `streamlit/` subdirectory)
- Handle missing data gracefully (try/except around queries, show "No data available" if tables don't exist yet)

### Custom Theme (`.streamlit/config.toml`)

```toml
[theme]
primaryColor = "#2E8BC0"
backgroundColor = "#FFFFFF"
secondaryBackgroundColor = "#F0F4F8"
textColor = "#1A3A5C"
font = "sans serif"
```

### Example Query Patterns

```python
# Get session
from snowflake.snowpark.context import get_active_session
session = get_active_session()

# Metric: total securities
count = session.sql("SELECT COUNT(*) FROM ORBIT_DEMO.CURATED.DIM_ISSUER").collect()[0][0]

# Latest stock prices
prices = session.sql("""
    SELECT TICKER, DATE, VALUE AS CLOSE_PRICE
    FROM ORBIT_DEMO.MARKET_DATA.FACT_STOCK_PRICES
    WHERE VARIABLE = 'post-market_close'
      AND TICKER = 'AAPL'
      AND DATE >= DATEADD('year', -1, CURRENT_DATE())
    ORDER BY DATE
""").to_pandas()

# Treasury yield curve (latest date)
yields = session.sql("""
    SELECT VARIABLE_NAME, VALUE
    FROM ORBIT_DEMO.MARKET_DATA.FACT_TREASURY_YIELDS
    WHERE DATE = (SELECT MAX(DATE) FROM ORBIT_DEMO.MARKET_DATA.FACT_TREASURY_YIELDS)
    ORDER BY VALUE
""").to_pandas()

# Company financials
financials = session.sql("""
    SELECT TAG, PERIOD_END_DATE, VALUE, UNIT
    FROM ORBIT_DEMO.MARKET_DATA.FACT_SEC_FINANCIALS
    WHERE CIK = '0000320193'  -- Apple
      AND STATEMENT = 'Income Statement'
      AND TAG IN ('Revenues', 'NetIncomeLoss', 'EarningsPerShareBasic')
    ORDER BY PERIOD_END_DATE DESC
    LIMIT 20
""").to_pandas()
```

### Key Points
- This is a PORTAL — it shows data and **deep-links to the AI agents in CoWork** via `st.link_button`. It does NOT embed agent chat (CoWork handles that natively with streaming, citations, and multi-turn memory).
- The CoWork URL pattern is: `https://<account>.snowflakecomputing.com/intelligence/cowork?agent=<DB>.<SCHEMA>.<AGENT_NAME>`
- Use `session.get_current_account()` to dynamically build the account URL — don't hardcode it.
- Focus on clean, professional presentation of real data.
- Every chart/metric should come from real tables in ORBIT_DEMO.
- Use Streamlit's native charting (st.line_chart, st.bar_chart) or plotly for more control.
- The app should load fast — use `@st.cache_data` with TTL for expensive queries.
- On every data page (Research, Portfolio, Market), include a contextual "Ask the Agent" link button that takes the user to the relevant agent in CoWork. E.g. on the Research page for Apple, show a button "Ask Research Agent about Apple →" linking to CoWork with the Research agent.

Build this as a complete, working application. Start with the landing page and add pages one at a time.

## END OF PROMPT

---

## Deployment Notes

After the Streamlit app is built:

1. **In-Workspace deployment (quickest):** The app runs directly in the workspace notebook service
2. **Streamlit in Snowflake (production):** Deploy via `CREATE STREAMLIT` DDL or `snow streamlit deploy`
3. **For the hub pattern Sahana wants:** Each dataset (SEC, Macro, Credit Card when Bloomberg arrives) can be a separate Streamlit page within this multi-page app, or separate apps linked from the portal

## Logo Files Needed

Save the following logo images into `assets/logos/` in the workspace:
- `orbit_logo_dark_square.png` — square ORBIT icon (neural network atom) on dark navy background
- `orbit_logo_light_square.png` — same icon on light/white background
- `orbit_logo_dark_horizontal.png` — horizontal layout with "ORBIT | Omnicient Reasoning Barclays Intelligence Tool" on dark
- `orbit_logo_light_horizontal.png` — same on light background

Brand colours extracted from the logos:
- Teal orbit rings: `#2E8BC0` to `#1B6B93` gradient
- Neural network nodes: `#5DADE2` (light blue)
- Dark background: `#1B2838`
- Light text: `#FFFFFF`
- Dark text: `#1A3A5C`
