# N8N + MF Engine API: One-Week Workshop

## Overview

This workshop takes you from zero to a production-ready AI-powered portfolio intelligence platform using N8N workflow automation and the MF Engine API. Over seven days you will build increasingly sophisticated workflows — starting with basic API calls, progressing through CAS parsing, portfolio analytics, stock screeners, MCP-powered AI agents, human-in-the-loop approval flows, and culminating in a fully evaluated, production-hardened system.

---

## Workshop Schedule

| Day | Title | Core Concept | Primary Build |
|-----|-------|-------------|---------------|
| 1 | N8N Fundamentals + MF APIs | HTTP requests, credentials, node wiring | Scheme Explorer (search → NAV → details) |
| 2 | CAS PDF Parser | Binary file upload, multipart POST, iteration | CAS Upload Pipeline → HTML portfolio report |
| 3 | Portfolio Xray & Analytics | Scheduled workflows, DB storage, alerting | Portfolio Health Dashboard with stress tests |
| 4 | Stoic Stocks + MF Integration | Cross-dataset joins, institutional alpha | Institutional Alpha Scanner + Fund-Stock mapper |
| 5 | MCP Integration in N8N | AI Agent node, SSE MCP servers, tool routing | AI Portfolio Advisor via MCP tools |
| 6 | Human-in-the-Loop Workflows | Wait node, webhook resume, approval gates | Smart Redemption Advisor with Human Gate |
| 7 | Pipeline Evaluation & Production | Evaluation patterns, error handling, monitoring | Full pipeline evaluator + capstone system |

---

## Prerequisites

### Technical Requirements
- N8N instance (self-hosted v1.40+ or n8n.cloud)
- Node.js 18+ (for local N8N)
- PostgreSQL or SQLite (for Day 3 storage)
- SMTP access or Gmail OAuth (for Day 2/6 email nodes)
- OpenAI API key (for Day 5/6/7 AI Agent nodes)
- Telegram Bot Token (optional, for Day 3 alerts)

### Knowledge Prerequisites
- Basic understanding of REST APIs and JSON
- Familiarity with JavaScript (for Code nodes)
- Basic understanding of mutual funds (NAV, XIRR, CAS)
- No prior N8N experience required

### Accounts to Create Before Starting
1. MF Engine API account — obtain your `x-api-key` from the partner dashboard
2. OpenAI account — get API key from platform.openai.com
3. SMTP credentials (Gmail App Password or SendGrid free tier)
4. Airtable account (for Day 7 storage)

---

## API Key Setup Instructions

### Step 1: Obtain Your MF Engine API Key
Contact the MF Engine partner team or sign up at the portal. You will receive an `x-api-key` value like `mfe_live_xxxxxxxxxxxxxxxx`.

### Step 2: Add Credential in N8N

1. Open N8N → **Settings** → **Credentials** → **Add Credential**
2. Search for **Header Auth**
3. Set:
   - Name: `MF Engine API`
   - Header Name: `x-api-key`
   - Header Value: `<your-api-key>`
4. Click **Save**

### Step 3: Use in HTTP Request Nodes
In any HTTP Request node, under **Authentication** → select **Generic Credential Type** → choose **MF Engine API**.

### Environment Notes
- Production base URL: `https://app2.mfapis.club/api/v2`
- Staging base URL: `https://staging-app.mfapis.club/api/v2`
- All routes are prefixed with `/api/v2/`
- Start with staging during development; switch to production for final demos

---

## MCP Server Setup in N8N

The MF Engine exposes four MCP (Model Context Protocol) servers over SSE (Server-Sent Events) transport. These integrate with N8N's **AI Agent** node to give the AI structured access to portfolio data.

### Available MCP Servers

| Server Path | Purpose | Key Tools |
|-------------|---------|-----------|
| `/mcp/portfolio-analytics` | Deep portfolio analysis | getPortfolioXray, getPortfolioHealthScore, getFundAnalytics, comparePortfolioFunds, getRedemptionRecommendations, getTaxAwareRedemptionPlan, getFundVsIndexComparison, getPortfolioAllocationAnalysis, getSipProjection, getSmartRedemptionPlan, getPortfolioFactorExposure, getPortfolioStressTest, getTopAlternativeFunds, getCasImportHistory |
| `/mcp/portfolio-cas` | CAS-based portfolio tools | getPortfolioAnalytics, getSchemeResearch, getBucketRecommendations, getRebalancePlan, stageBucketPurchaseCart |
| `/mcp/investor` | Investor-facing tools | Investor account, holdings, transactions |
| `/mcp/partner` | Partner-facing tools | Partner dashboard, client management |

### Connecting MCP in N8N AI Agent

1. Add an **AI Agent** node to your workflow
2. In the AI Agent, click **Add Tool** → **MCP Client**
3. Configure the MCP Client tool:
   - Transport: **SSE**
   - URL: `https://app2.mfapis.club/mcp/portfolio-analytics`
   - Headers: `x-api-key: <your-api-key>`
4. The AI Agent will automatically discover all tools from that MCP server
5. Repeat for additional MCP servers as needed

### System Prompt Guidance for MCP
When configuring the AI Agent system prompt, specify which MCP tools to prefer for different query types:
- Portfolio health questions → `getPortfolioHealthScore` then `getPortfolioXray`
- Redemption queries → `getSmartRedemptionPlan` or `getTaxAwareRedemptionPlan`
- SIP planning → `getSipProjection`
- Fund comparison → `comparePortfolioFunds` or `getTopAlternativeFunds`

---

## Learning Objectives

By the end of the seven-day workshop you will be able to:

1. **Build and deploy N8N workflows** integrating REST APIs with proper authentication, error handling, and data transformation
2. **Parse real CAS PDFs** and extract structured portfolio holdings with XIRR calculations
3. **Schedule automated portfolio health checks** that alert advisors when metrics fall below thresholds
4. **Combine MF disclosure data with stock screener data** to generate institutional alpha signals
5. **Deploy AI agents with MCP tools** that can answer complex portfolio questions by calling the right analytical tools
6. **Implement human-in-the-loop approval gates** for regulatory compliance in financial workflows
7. **Evaluate AI pipeline quality** using reference datasets, rubric scoring, and regression testing
8. **Harden workflows for production** with retry logic, dead-letter queues, structured logging, and SLA monitoring

---

## What Each Day Builds

### Day 1 — Scheme Explorer
A webhook-triggered workflow that accepts a fund name as input, searches the MF universe, fetches the current NAV and historical price data, and returns a formatted fund profile. Introduces all core N8N nodes.

### Day 2 — CAS Upload Pipeline
An end-to-end pipeline that accepts a CAS PDF, parses holdings using the MF Engine parser API, computes portfolio totals (invested, current value, overall XIRR), generates an HTML report, and emails it to an advisor for review before storing the data.

### Day 3 — Portfolio Health Dashboard
A scheduled (daily) workflow that runs portfolio Xray analysis, computes a health score, performs stress tests against historical scenarios, stores the time-series data in a database, and sends Telegram alerts when the health score drops below a configurable threshold.

### Day 4 — Institutional Alpha Scanner
A dual-stream workflow that fetches AMC portfolio disclosures (which stocks funds are buying) and the Stoic stock screener (RSI, MACD, technicals), then joins the two datasets to identify stocks held by three or more funds AND currently oversold (RSI < 40). Also builds a reverse lookup: given a stock, find all funds that hold it.

### Day 5 — AI Portfolio Advisor
A webhook-triggered AI Agent workflow where an investor or advisor can ask a natural language question ("Should I switch from HDFC Mid Cap to SBI Magnum?") and the AI Agent uses MCP tools to research the answer, returning a structured recommendation with supporting data.

### Day 6 — Smart Redemption Advisor with Human Gate
An AI-driven redemption planner that generates a tax-aware redemption plan, pauses the workflow, sends an email to a compliance officer with Approve/Reject links, resumes on the officer's decision, and either executes the plan or logs the rejection. Includes 24-hour timeout escalation.

### Day 7 — Pipeline Evaluator + Production Hardening
A comprehensive quality assurance workflow that runs five predefined test queries through the AI advisor, evaluates each response against reference answers using a rubric (accuracy, completeness, tool selection), stores scores in Airtable, and generates a quality report. Covers retry logic, dead-letter webhooks, and structured logging.

---

## Capstone Project

**The AI Portfolio Intelligence Platform** combines all seven days into a unified system:

```
Investor CAS Upload (Day 2)
         ↓
Portfolio Xray + Health Score (Day 3)
         ↓
Alpha Signal Overlay (Day 4) 
         ↓
AI Advisor generates recommendations (Day 5)
         ↓
Compliance Human Gate (Day 6)
         ↓
Approved recommendation delivered to investor
         ↓
Quality logged and evaluated (Day 7)
```

The capstone demonstrates:
- Real CAS data ingestion and parsing
- Daily automated health monitoring
- Institutional signal overlay on holdings
- Natural language AI advisory capability
- Regulatory-compliant approval workflow
- Production quality metrics and alerting

Participants will demo their capstone on Day 7 afternoon.

---

## File Structure

```
n8n-workshop-files/
├── WEEK_WORKSHOP.md          # This master guide
├── days/
│   ├── day1_guide.md         # N8N Fundamentals + MF APIs
│   ├── day2_guide.md         # CAS PDF Parser
│   ├── day3_guide.md         # Portfolio Xray & Analytics
│   ├── day4_guide.md         # Stoic Stocks + MF Integration
│   ├── day5_guide.md         # MCP Integration in N8N
│   ├── day6_guide.md         # Human-in-the-Loop Workflows
│   └── day7_guide.md         # Pipeline Evaluation & Production
├── workflows/
│   ├── day1_scheme_explorer.json
│   ├── day2_cas_parser.json
│   ├── day3_portfolio_xray.json
│   ├── day4_alpha_scanner.json
│   ├── day5_ai_advisor.json
│   ├── day6_human_approval.json
│   └── day7_pipeline_evaluator.json
└── extras/
    └── (supplementary materials)
```

---

## Quick Start Checklist

Before Day 1:
- [ ] N8N instance running and accessible
- [ ] MF Engine API key obtained and added as credential
- [ ] OpenAI API key added as credential in N8N
- [ ] SMTP credentials configured
- [ ] Imported `day1_scheme_explorer.json` into N8N
- [ ] Tested the webhook URL with a Postman/curl call
- [ ] Read through Day 1 guide

---

## API Reference Quick Card

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/scheme?search=hdfc&limit=10` | GET | Search fund schemes |
| `/scheme/:scheme_id` | GET | Single scheme details |
| `/nav?scheme_id=UUID&limit=30` | GET | Historical NAV |
| `/cas` | POST multipart | Parse CAS PDF |
| `/amc_factsheet?amc_name=HDFC` | GET | AMC factsheets |
| `/amc_portfolio_disclosure?month=2025-05` | GET | Fund equity holdings |
| `/nse_index?symbol=NIFTY+50` | GET | NSE index OHLCV |
| `/mf_query?category=Large+Cap` | GET | Query MF universe |
| `/stocks?type=equity_screener` | GET | Stoic stock screener |
| `/stocks?type=fno_screener` | GET | F&O screener |
| `/stocks?type=movers` | GET | Top movers/losers |
| `/stocks/ohlcv?symbol=RELIANCE` | GET | Stock OHLCV |
| `/stocks/fundamentals?symbol=RELIANCE` | GET | PE, PB, ROE, market cap |
| `/stocks/financials?symbol=RELIANCE&period=quarterly` | GET | Revenue, EBITDA, EPS |
| `/stocks/sector_peers?symbol=RELIANCE` | GET | Sector peers |
| `/stocks/search?q=reliance` | GET | Search stocks |
