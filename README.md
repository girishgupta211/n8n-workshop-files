# N8N Workshop — AI-Powered Mutual Fund Intelligence Platform

A one-week, hands-on workshop for building production-grade agentic financial workflows with N8N and the MF Engine API (`https://app2.mfapis.club`).

---

## Workshop Structure

| Day | Guide | Workflow | What You Build |
|-----|-------|----------|----------------|
| 1 | [day1_guide.md](days/day1_guide.md) | [day1_scheme_explorer.json](workflows/day1_scheme_explorer.json) | Scheme Explorer — search funds + live NAV |
| 2 | [day2_guide.md](days/day2_guide.md) | [day2_cas_parser.json](workflows/day2_cas_parser.json) | CAS PDF Parser + advisor approval gate |
| 3 | [day3_guide.md](days/day3_guide.md) | [day3_portfolio_xray.json](workflows/day3_portfolio_xray.json) | Portfolio Health Dashboard (scheduled, Telegram alerts) |
| 4 | [day4_guide.md](days/day4_guide.md) | [day4_alpha_scanner.json](workflows/day4_alpha_scanner.json) | Institutional Alpha Scanner (funds × Stoic screener) |
| 5 | [day5_guide.md](days/day5_guide.md) | [day5_ai_advisor.json](workflows/day5_ai_advisor.json) | AI Portfolio Advisor via MCP |
| 6 | [day6_guide.md](days/day6_guide.md) | [day6_human_approval.json](workflows/day6_human_approval.json) | Smart Redemption + Compliance Gate |
| 7 | [day7_guide.md](days/day7_guide.md) | [day7_pipeline_evaluator.json](workflows/day7_pipeline_evaluator.json) | Pipeline Evaluator + Production Hardening |

**Master overview:** [WEEK_WORKSHOP.md](WEEK_WORKSHOP.md)

---

## Quick Start

### Prerequisites
- n8n (cloud: [app.n8n.cloud](https://app.n8n.cloud) or local: `npx n8n`)
- MF Engine API key — contact your workshop coordinator
- MCP Auth Token — needed from Day 5
- OpenAI API key — needed from Day 5
- A CAMS or KFintech CAS PDF — needed for Day 2

### Credentials to configure in n8n

| Credential Name | Type | Header / Field | Value |
|----------------|------|---------------|-------|
| `MF Engine API Key` | Header Auth | `x-api-key` | your API key |
| `MF Engine MCP Token` | Header Auth | `Authorization` | `Bearer <mcp-token>` |
| `OpenAI` | OpenAI API | API Key | `sk-...` |

### n8n Variable to set

Go to **Settings → Variables** and add:
```
MF_API_BASE  =  https://app2.mfapis.club/api/v2
```

### Import a workflow

1. Open n8n → **Workflows** → **Import from file**
2. Select any `workflows/day*.json` file
3. Configure credentials when prompted
4. Click **Execute** or activate the workflow

---

## API Reference (Quick)

```
Base URL: https://app2.mfapis.club/api/v2
Auth:     x-api-key: <your-key>

GET  /scheme?search=hdfc&limit=10           Scheme search
GET  /nav?scheme_id=UUID&limit=1            Latest NAV
POST /cas                                   Parse CAS PDF (multipart)
GET  /amc_portfolio_disclosure?from=2025-05 Monthly fund equity holdings
GET  /stocks/screener/equity?limit=200      Stoic equity screener
GET  /stocks/screener/fno?limit=100         F&O screener
GET  /stocks/screener/movers?limit=20       Top movers/losers
```

## MCP Servers (Day 5+)

```
POST https://app2.mfapis.club/mcp/portfolio-analytics
POST https://app2.mfapis.club/mcp/portfolio-cas
POST https://app2.mfapis.club/mcp/stocks
POST https://app2.mfapis.club/mcp/nse-index
POST https://app2.mfapis.club/mcp/investor-support
POST https://app2.mfapis.club/mcp/partner-investor-actions

Auth:  Authorization: Bearer <mcp-token>
       x-actor-role: PARTNER   (workshop testing bypass)
       x-actor-id: <partner-id>
```

---

## Advanced Workshop (Multi-Agent System)

See the [`extras/`](extras/) folder for a 6-workflow multi-agent system:
- [WORKSHOP_GUIDE_ADVANCED.md](extras/WORKSHOP_GUIDE_ADVANCED.md) — full setup guide
- `main_orchestrator.json` — central AI orchestrator with specialist sub-agents
- `subagent_midcap_analyst.json` / `subagent_psu_analyst.json` — domain-specialist agents
- `subworkflow_data_layer.json` — data fetching with fallback
- `subworkflow_prepare_trade.json` — human-in-the-loop compliance gate
- `subworkflow_error_handler.json` — error formatter

---

## Disclaimer

All workflows and recommendations are for educational purposes only. Nothing in this workshop constitutes financial advice. Past performance of mutual funds does not guarantee future results.