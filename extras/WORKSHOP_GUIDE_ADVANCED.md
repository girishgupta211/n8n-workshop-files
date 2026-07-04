# 🤖 Build a Multi-Agent Financial AI System
## Advanced Workshop Guide — No Coding Experience Needed

> **What you'll build:** A fully automated AI system where multiple AI agents work as a team to research mutual funds, analyze risk, generate investment reports, and send them via email and Slack — all triggered by a single question typed in a chat.

---

## 🎯 What Does This System Do?

Imagine you type:

> *"Should I invest in mid-cap funds right now?"*

Here is what happens automatically in the next 10 seconds:

```
You type a question
        ↓
System checks: Is this a valid investment question?
        ↓
Fetches live mutual fund data from the API
        ↓
Decides: Is this about Mid-Cap or PSU funds?
        ↓
Sends the question to the right specialist AI agent
        ↓
Agent calculates CAGR and volatility
        ↓
If the recommendation involves high risk → asks a human to approve
        ↓
Generates a full investment report
        ↓
Sends the report to your Email AND Slack
        ↓
Shows you the final answer
```

This is what a **real agentic AI system** looks like — not just a chatbot that answers questions, but a system that **takes action** based on reasoning.

---

## 🧠 Key Concepts You Will Learn

| Concept | Simple explanation | Where you see it |
|---|---|---|
| **AI Agent** | An AI that thinks, decides, and acts — not just answers | Financial Orchestrator, Midcap Analyst, PSU Analyst |
| **Multi-Agent System** | Multiple AI specialists working as a team | Orchestrator calls Midcap or PSU agent depending on the question |
| **Tool Use** | AI using external functions like a calculator | CAGR Calculator, Volatility Calculator tools |
| **Memory** | AI remembers the conversation history | Simple Memory node |
| **Human in the Loop** | A human approves risky decisions before they execute | Wait for Reviewer node in Prepare Trade |
| **Subworkflow** | A mini-workflow called by a parent workflow | All 5 supporting workflows |
| **Guardrails** | Rules that prevent the AI from doing harmful things | Validate Query Input, If Query Valid nodes |

---

## 📐 The Full System Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    MAIN ORCHESTRATOR                     │
│                                                         │
│  You → [Webhook] → [Validate] → [Fetch Fund Data]      │
│                          ↓                             │
│                   [AI Orchestrator]                     │
│                   (decides routing)                     │
│                    ↙         ↘                          │
│          [Midcap Agent]    [PSU Agent]                  │
│                    ↘         ↙                          │
│              [Format Report]                            │
│              [Email] + [Slack]                          │
└─────────────────────────────────────────────────────────┘

Supporting workflows (called automatically):
├── Data Layer      → fetches live fund data
├── Prepare Trade   → handles human approval for risky calls
└── Error Handler   → catches and reports any failures
```

---

## 📁 The 6 Files and What Each Does

### 1. `main_orchestrator.json` — The Brain
**Role:** The central coordinator. Receives your question, fetches data, decides which specialist to call, formats the final report, and sends it out.

**Think of it as:** The senior portfolio manager who receives a client question, gathers research, delegates to analysts, and presents the final recommendation.

---

### 2. `subworkflow_data_layer.json` — The Data Fetcher
**Role:** Fetches live mutual fund data. First tries the premium MfAPIs data source (rich data). If that fails, falls back to the free public API. Always returns a clean, standardized result.

**Think of it as:** The research library. The main office calls the library, which finds the best available data and hands it back in a consistent format.

**Key design pattern — Fallback:**
```
Try Premium API first
    ↓ (if it works)
Return rich data with NAV history, factsheet, holdings
    ↓ (if it fails)
Try free public API as backup
    ↓
Return basic data — still useful, just less detailed
```
This makes the system **resilient** — it never completely fails just because one data source is unavailable.

---

### 3. `subagent_midcap_analyst.json` — Midcap Specialist AI
**Role:** A dedicated AI analyst that only handles Mid-Cap fund questions. Has access to two tools:
- **CAGR Calculator** — computes compound annual growth rate from NAV history
- **Volatility Calculator** — measures how risky/stable the fund's returns have been

After analysis, if it recommends action → it calls the Trade Preparation workflow for approval.

**Think of it as:** A junior analyst who specializes only in mid-cap stocks. Knows this area deeply, uses specific analytical tools, and escalates trade ideas to compliance before execution.

---

### 4. `subagent_psu_analyst.json` — PSU Specialist AI
**Role:** Identical structure to the Midcap Analyst, but specializes in Public Sector Undertaking (government-owned company) funds. Different AI persona, different focus area, same tools.

**Think of it as:** A second junior analyst specializing in government-backed investments.

---

### 5. `subworkflow_prepare_trade.json` — Human Approval Gate
**Role:** Before any high-risk investment action is taken, this workflow:
1. Checks if the recommendation is high risk
2. If **low risk** → auto-approves immediately
3. If **high risk** → sends an email to a reviewer and **pauses** to wait for their response
4. Resumes only after the reviewer clicks Approve or Reject

**Think of it as:** The compliance department. Every trade idea must pass through compliance. Small trades auto-clear. Large or risky trades wait for a human sign-off.

**This is "Human in the Loop" AI** — the system knows its own limits and asks for help when the stakes are high.

---

### 6. `subworkflow_error_handler.json` — The Safety Net
**Role:** If anything goes wrong anywhere in the system, this catches the error, formats it cleanly, and reports it — instead of crashing silently.

**Think of it as:** The alarm system. If a fire breaks out in any part of the building, the alarm triggers, tells you exactly which room, and calls the right people.

---

## 📋 Step-by-Step Build Guide

---

### PHASE 1 — Import All Workflows (10 minutes)

> In n8n, workflows can call other workflows. You must import all 6 files before activating anything.

**Import order matters:**

| Step | File to import | Why first |
|---|---|---|
| 1 | `subworkflow_error_handler.json` | Other workflows reference this |
| 2 | `subworkflow_data_layer.json` | Data fetcher needed by orchestrator |
| 3 | `subworkflow_prepare_trade.json` | Needed by the analyst agents |
| 4 | `subagent_midcap_analyst.json` | Needed by orchestrator |
| 5 | `subagent_psu_analyst.json` | Needed by orchestrator |
| 6 | `main_orchestrator.json` | Import last — references all others |

**To import each file:**
1. n8n → click **"+"** (New Workflow) → **"Import from file"**
2. Select the JSON file
3. Click **Save**
4. Do NOT activate yet — import all 6 first

---

### PHASE 2 — Connect the Workflows (5 minutes)

After importing, the orchestrator needs to know the IDs of the subworkflows it calls.

1. Open **Main Orchestrator** workflow
2. Find the **"Workflow - Midcap Analyst"** node → click it → set the workflow to `Subworkflow - Midcap Analyst`
3. Find the **"Workflow - PSU Analyst"** node → set to `Subworkflow - PSU Analyst`
4. Find the **"Fetch Fund Data"** node → set to `Subworkflow - Data Layer`
5. Save the workflow

Similarly in the analyst subworkflows:
- Open **Subworkflow - Midcap Analyst** → find **"Workflow - Prepare Trade"** → set to `Subworkflow - Prepare Trade`
- Open **Subworkflow - PSU Analyst** → find **"Workflow - Prepare Trade"** → set to `Subworkflow - Prepare Trade`

---

### PHASE 3 — Add Your API Key (3 minutes)

> The system uses MfAPIs to get live mutual fund data.

1. In n8n → **Settings** → **Credentials** → **New Credential** → **HTTP Header Auth**
2. Name: `MfAPIs Key`
3. Header name: `x-api-key`
4. Header value: `YOUR_API_KEY_HERE` *(get your key from contact@mfapis.in)*
5. Save

Then in **Subworkflow - Data Layer**, update the HTTP Request nodes to use this credential.

---

### PHASE 4 — Add OpenAI Key (2 minutes)

The AI agents (Orchestrator, Midcap Analyst, PSU Analyst) use OpenAI GPT-4.

1. n8n → **Settings** → **Credentials** → **New** → **OpenAI API**
2. Paste your OpenAI API key
3. All three AI Agent nodes will automatically use it

---

### PHASE 5 — Configure Notifications (optional, 5 minutes)

**For Email reports:**
1. Open **Main Orchestrator** → click **"Send Email Report"** node
2. Add your SMTP credentials
3. Set the **To** email address

**For Slack reports:**
1. Open **Main Orchestrator** → click **"Send Slack Report"** node
2. Connect your Slack workspace
3. Set the channel name (e.g. `#mf-research`)

---

### PHASE 6 — Activate (2 minutes)

Activate in this order:

1. `Subworkflow - Error Handler` → toggle **Active**
2. `Subworkflow - Data Layer` → toggle **Active**
3. `Subworkflow - Prepare Trade` → toggle **Active**
4. `Subworkflow - Midcap Analyst` → toggle **Active**
5. `Subworkflow - PSU Analyst` → toggle **Active**
6. `Main Orchestrator` → toggle **Active** ← last

---

### PHASE 7 — Test It

Send a POST request to your webhook URL with a question:

```bash
curl -X POST http://localhost:5678/webhook/financial-analyst \
  -H "Content-Type: application/json" \
  -d '{"query": "Analyze the top mid cap mutual fund for me"}'
```

Or use a tool like **Postman** or **Insomnia** if you prefer a UI.

**Expected flow:**
1. Orchestrator receives your question
2. Fetches live fund data
3. Routes to Midcap Analyst (because you said "mid cap")
4. Analyst calculates CAGR and volatility
5. Formats a full investment report
6. Sends to email + Slack
7. Returns the report in the API response

---

## 🔍 Reading the Output

The system returns a structured report like this:

```
📊 MUTUAL FUND ANALYSIS REPORT
================================
Fund: HDFC Mid-Cap Opportunities Fund
Query: Analyze top mid cap fund

📈 PERFORMANCE
  3-Year CAGR: 22.4%
  5-Year CAGR: 18.7%

⚡ RISK METRICS
  Volatility (1Y): 14.2%
  Risk Level: MODERATE

🎯 ANALYST RECOMMENDATION
  The fund has delivered consistent above-benchmark returns
  with moderate volatility. Suitable for investors with a
  3+ year horizon who can tolerate short-term fluctuations.

⚠️ RISK DISCLOSURE
  Past performance is not indicative of future results.
  This is not financial advice.
```

---

## 🔄 How the Agents Communicate

```
Main Orchestrator
│
│ "Here is the fund data and user question.
│  Analyze the mid-cap opportunity."
│
↓ calls ────────────────────────────────────
Midcap Analyst Agent
│
│ [uses cagr_calculator tool]
│   Input: NAV history array
│   Output: "3-year CAGR = 22.4%"
│
│ [uses volatility_calculator tool]
│   Input: NAV history array
│   Output: "Volatility = 14.2%"
│
│ "Based on my analysis, I recommend..."
│
↓ returns ──────────────────────────────────
Main Orchestrator
│
│ Formats report → sends email + Slack
│ Returns response to user
```

This is a **ReAct agent pattern** (Reason + Act):
- The AI **reasons** about what tools it needs
- **acts** by calling those tools
- **reasons again** about the results
- **acts** by forming its recommendation

---

## 🛡️ Guardrails in This System

The system has 4 layers of protection:

| Layer | Node | What it prevents |
|---|---|---|
| **Input validation** | Validate Query Input | Gibberish, off-topic questions, injection attacks |
| **Data validation** | If Data Valid | Acting on empty or corrupt fund data |
| **Risk gate** | Prepare Trade → If High Risk | Executing high-risk recommendations without human review |
| **Error catching** | Error Handler | Silent failures that corrupt data downstream |

---

## 💡 The Difference Between a Chatbot and an Agent

| Chatbot | This AI Agent System |
|---|---|
| Answers questions | Takes action based on answers |
| Uses only its training | Uses live real-time data |
| Single response | Multi-step reasoning with tools |
| No memory | Remembers conversation context |
| No oversight | Human approval for high-risk decisions |
| Responds once | Sends email, Slack, and API response |

---

## 🚀 Ideas to Extend This System

1. **Add a Small-Cap Analyst** — duplicate the Midcap Analyst, change the persona to "Small Cap Specialist"
2. **Add WhatsApp notifications** — replace or add alongside Slack
3. **Add a portfolio tracker** — instead of one fund, analyze your full portfolio monthly
4. **Schedule it** — run every month on the 1st using n8n's Schedule trigger
5. **Add a voice interface** — connect to a speech-to-text API to ask questions verbally
6. **Connect to BSE StarMF** — extend the trade preparation to actually place orders

---

## ❓ Common Questions

**Q: What if I don't have an OpenAI API key?**
Replace the OpenAI node with any supported LLM in n8n — Anthropic Claude, Google Gemini, or Mistral all work. Claude is highly recommended for financial analysis.

**Q: What if the email/Slack nodes fail?**
The system still works — email and Slack are notifications only. The API response still returns the full report.

**Q: Can I add more than 2 specialist agents?**
Yes. Duplicate any analyst subworkflow, change the system prompt, and add a new `toolWorkflow` node to the orchestrator. The orchestrator AI will automatically learn to route to it based on the tool description.

**Q: What does "Human in the Loop" mean practically?**
When the AI recommends a high-risk action, an email is sent to a reviewer with Approve/Reject buttons. The entire system pauses and waits — it will not proceed until a human responds. This is the gold standard for responsible AI in financial systems.

---

## 📁 File Reference

| File | Role | Must be active? |
|---|---|---|
| `main_orchestrator.json` | Central brain | ✅ Yes |
| `subworkflow_data_layer.json` | Fetches fund data | ✅ Yes |
| `subagent_midcap_analyst.json` | Midcap AI specialist | ✅ Yes |
| `subagent_psu_analyst.json` | PSU AI specialist | ✅ Yes |
| `subworkflow_prepare_trade.json` | Human approval gate | ✅ Yes |
| `subworkflow_error_handler.json` | Catches errors | ✅ Yes |

---

## ⚠️ Disclaimer

All strategies and analyses shown are for **educational purposes only**.
This system does not provide financial advice.
Past performance is not indicative of future results.
Always consult a SEBI-registered investment advisor before making investment decisions.

---

*Built with [n8n](https://n8n.io) + [MfAPIs](https://mfapis.in) · Advanced Workshop by MfAPIs*
