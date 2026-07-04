# 🏦 Build a Financial AI Analyst Dashboard
## Step-by-Step Workshop Guide

> **What you'll build:** A live dashboard that reads Indian mutual fund portfolio disclosures and surfaces 4 institutional-grade investment signals — all without writing backend code.

---

## 🎯 What is the Outcome?

By the end of this workshop, opening one URL in your browser will show you:

| Panel | What it tells you |
|---|---|
| **Conviction Clustering** | Stocks that 2+ fund managers independently hold heavily — cross-validated institutional bets |
| **Institutional Alpha Basket** | A ready-to-use equal-weight portfolio built from the top consensus picks |
| **Sector Rotation Heatmap** | Which sectors are receiving inflows vs being distributed this month |
| **New Top-10 Momentum** | Stocks freshly added to multiple fund portfolios — fresh institutional bets |

This is the kind of intelligence Bloomberg Terminal and Morningstar Direct charge thousands of dollars for. You'll build it in 45 minutes.

---

## 🧠 Concepts You'll Learn

| Concept | Where it appears |
|---|---|
| **What is an API?** | Every HTTP Request node calling `app2.mfapis.club` |
| **What is a Webhook?** | The trigger node that listens for browser requests |
| **What is an AI Agent?** | The logic in Code nodes that reasons over data |
| **LLM vs Agentic AI** | Code nodes = deterministic agent; AI node = LLM-powered agent |
| **Data → Signal → Action** | The full pipeline from raw holdings to investment signals |

---

## 🛠️ Prerequisites

- n8n account (cloud or local)
- MfAPIs key (get yours at [mfapis.in](https://mfapis.in) or contact@mfapis.in)
- OpenAI API key (required only for the AI Agent extension in Step 10+)
- A browser

No coding experience needed. All logic is pre-written — you just connect the nodes.

---

## 🔐 Setting Up Credentials

> Do this **before** building the workflow. Credentials are stored once in n8n and reused across all nodes.

---

### Credential 1 — MfAPIs Key (required)

This key authenticates every HTTP Request node that calls `app2.mfapis.club`.

1. In n8n, click your **profile icon** (bottom-left) → **Settings**
2. Go to **Credentials** → click **+ Add credential**
3. Search for **"Header Auth"** → select it
4. Fill in:
   - **Credential name**: `MfAPIs Key`
   - **Name**: `x-api-key`
   - **Value**: *(paste your MfAPIs key here)*
5. Click **Save**

**How to use it in an HTTP Request node:**
- Open any HTTP Request node
- Scroll to **Authentication** → set to **"Predefined Credential Type"** or **"Generic Credential Type"**
- Select **Header Auth** → choose `MfAPIs Key`

> Alternatively: in each HTTP Request node → **Headers** tab → add manually:
> - Header name: `x-api-key`
> - Header value: *(paste your key directly)*

---

### Credential 2 — OpenAI API Key (for AI Agent extension)

Required only if you add an **AI Agent node** (e.g., to generate plain-English explanations of signals).

1. In n8n → **Settings** → **Credentials** → **+ Add credential**
2. Search for **"OpenAI"** → select it
3. Fill in:
   - **Credential name**: `OpenAI Key`
   - **API Key**: *(paste your OpenAI API key)*
   - Leave **Organization ID** blank unless your account requires it
4. Click **Save**

**How to get an OpenAI API key:**
1. Go to [platform.openai.com](https://platform.openai.com)
2. Sign in → click your profile → **API keys**
3. Click **+ Create new secret key** → copy it immediately (shown only once)

> **No OpenAI key?** You can substitute any n8n-supported LLM — Anthropic Claude, Google Gemini, or Mistral. Claude is recommended for financial analysis. Setup steps are identical: search for the provider name when adding a credential.

---

## 📐 The Flow Architecture

```
Browser opens URL
       ↓
 [Webhook Trigger]        ← listens at /webhook/mf-alpha-dashboard
       ↓
 [Fetch Large Cap Fund]   ← API call 1: find biggest Large Cap fund
 [Fetch Flexi Cap Fund]   ← API call 2: find biggest Flexi Cap fund
 [Fetch Mid Cap Fund]     ← API call 3: find biggest Mid Cap fund
       ↓
 [Extract Fund IDs]       ← picks ID + ISIN from each response
       ↓
 [Holdings: Large Cap]    ← API call 4: all stocks this fund holds + changes
 [Holdings: Flexi Cap]    ← API call 5: all stocks this fund holds + changes
 [Holdings: Mid Cap]      ← API call 6: all stocks this fund holds + changes
       ↓
 [Compute All Signals]    ← runs 4 strategy algorithms on the data
       ↓
 [Generate Dashboard HTML]← builds a Chart.js webpage from the signals
       ↓
 [Respond with Dashboard] ← sends the HTML back to your browser
```

**Total: 6 live API calls → 4 strategies computed → 1 live dashboard**

---

## 📋 Step-by-Step Build Guide

---

### STEP 1 — Create a New Workflow

1. Open n8n → click **"New Workflow"**
2. Name it: `MF Alpha Intelligence Dashboard`

---

### STEP 2 — Add the Webhook Trigger

> This node makes your workflow accessible via a URL.

1. Click **"+"** → search **"Webhook"** → select it
2. Set **HTTP Method** = `GET`
3. Set **Path** = `mf-alpha-dashboard`
4. Set **Respond** = `Using Respond to Webhook Node`
5. You'll see two URLs appear:
   - **Test URL**: works only while editor is open + you click Execute
   - **Production URL**: works always once workflow is Active ✅

---

### STEP 3 — Fetch the Three Funds (3 HTTP Request Nodes)

> Each node calls our MF Data API to find the top fund in a category.

Add **3 HTTP Request nodes** in sequence. For each:

**Node A — Large Cap**
- Method: `GET`
- URL: `https://app2.mfapis.club/api/v2/public/scheme`
- Headers: `x-api-key` = *(your API key)*
- Query params:
  - `q` = `large cap`
  - `plan` = `Direct`
  - `option` = `Growth`
  - `sort_by` = `aum`
  - `sort_dir` = `desc`
  - `limit` = `1`

**Node B — Flexi Cap** (same settings, `q` = `flexi cap`)

**Node C — Mid Cap** (same settings, `q` = `mid cap`)

> 💡 **Why sequential?** n8n processes one node at a time in this workflow. We chain them so each result is available when the next node runs.

---

### STEP 4 — Extract Fund IDs (Code Node)

> This node reads the 3 API responses and pulls out each fund's ID and ISIN.

1. Add a **Code** node
2. Paste this JavaScript:

```javascript
function pick(node) {
  const list = $(node).first().json?.data?.list ?? [];
  const f = list[0];
  const short = f?.name?.split(' - ')[0] ?? f?.name ?? 'Unknown';
  return f ? { id: f.id, isin: f.scheme_isin, name: f.name, short } : null;
}

return [{
  json: {
    largeCap: pick('Fetch Large Cap Funds'),
    flexiCap: pick('Fetch Flexi Cap Funds'),
    midCap:   pick('Fetch Mid Cap Funds')
  }
}];
```

> 💡 **What is `$('node name')`?** In n8n, this is how one node reads data from a previous node by name. It's the core of n8n's data flow.

---

### STEP 5 — Fetch Holdings for Each Fund (3 More HTTP Request Nodes)

> These nodes fetch the complete list of stocks each fund holds, including how weights changed month-over-month.

**Node D — Large Cap Holdings**
- URL: `https://app2.mfapis.club/api/v2/public/amc_portfolio_disclosure/scheme/{{ $('Extract Fund IDs').first().json.largeCap.isin }}/holding-changes`
- Header: `x-api-key` = *(your API key)*

**Node E — Flexi Cap Holdings** (replace `largeCap` with `flexiCap`)

**Node F — Mid Cap Holdings** (replace `largeCap` with `midCap`)

> 💡 **What is `{{ expression }}`?** n8n's way of injecting dynamic values into URLs. Here we're using the ISIN we extracted in Step 4.

---

### STEP 6 — Compute All 4 Signals (Code Node)

> This is the brain of the pipeline. One code node runs all 4 investment strategies.

Add a **Code** node and paste the signal computation logic (provided in `mf_four_alpha_strategies.json`).

**What each strategy does inside this node:**

**Strategy 1 — Conviction Clustering**
```
For each stock:
  Count how many funds hold it at ≥ 2% weight
  If 2+ funds → it's a conviction signal
  Score = avg_weight × number_of_funds
```

**Strategy 2 — Institutional Alpha Basket**
```
For every stock across all 3 funds:
  Score = total_weight_across_funds × number_of_funds
  Take top 20 → assign 5% equal weight each
  This is your synthetic institutional portfolio
```

**Strategy 3 — Sector Flow**
```
For each holding:
  absolute_change = current_weight - previous_weight
  Group by sector → sum all changes
  Positive = sector receiving inflows
  Negative = sector being distributed
```

**Strategy 4 — New Top-10 Momentum**
```
Filter holdings where status = "added"
  (status is pre-computed by our API — compares this month vs last month)
Group by stock name
  If 2+ funds independently added the same stock → signal fired
```

---

### STEP 7 — Generate the Dashboard HTML (Code Node)

> This node takes the computed signals and builds a complete webpage with interactive charts.

Add a **Code** node that builds an HTML string containing:
- **Chart.js** for bar charts and doughnut chart (loaded from CDN)
- **CSS** for the dark theme layout
- **JavaScript** that renders charts from the signal data

> 💡 **Key learning:** n8n Code nodes can generate any text output — including full HTML pages. This is how you build dashboards without a separate frontend.

> ⚠️ **Important:** Do NOT use backtick template literals (`` ` ``) inside the `<script>` block of your HTML. n8n's code runner cannot handle nested template literals. Use string concatenation (`'string' + variable + 'string'`) instead.

---

### STEP 8 — Respond with Dashboard (Respond to Webhook Node)

> This sends your HTML page back to whoever opened the URL.

1. Add a **Respond to Webhook** node
2. Set **Respond With** = `Text`
3. Set **Response Body** = `{{ $json.html }}`
4. Add **Response Header**: `Content-Type` = `text/html; charset=utf-8`

---

### STEP 9 — Activate and Open

1. Click the **Active toggle** (top-right of editor) → turns green
2. Open in your browser:
   ```
   http://localhost:5678/webhook/mf-alpha-dashboard
   ```
3. Wait ~5 seconds → your live dashboard appears 🎉

---

## 📊 Reading the Dashboard

### Strategy 1: Conviction Clustering
```
ICICI Bank  ████████████████████  Score: 20 | 3 funds | Confidence: 93/100
HDFC Bank   ████████████████      Score: 16 | 3 funds | Confidence: 87/100
```
- **Blue bar** = Conviction Score (higher = stronger cross-fund agreement)
- **Green bar** = Confidence Score (accounts for position size + ranking)
- **3 funds** = all 3 AMCs hold this stock heavily → very strong signal

### Strategy 2: Institutional Alpha Basket
```
Doughnut chart with 10 equal slices
Each slice = one stock = 5% of portfolio
```
- This is a ready-to-use portfolio
- Rebalance on the **15th of each month** (disclosure lag from AMFi)
- Higher conviction = better ranking = included in basket

### Strategy 3: Sector Rotation Heatmap
```
Transport Services  ████████████████  +4.18%  ← AMCs buying
Finance             ████████████       +4.07%  ← AMCs buying
IT - Software       ▁                  -0.09%  ← AMCs selling
```
- **Green bars = inflow**: fund managers actively increasing allocation
- **Red bars = outflow**: fund managers reducing or exiting
- Typically **leads market by 2-6 weeks** — use as a forward indicator
- Drivers below each bar show which specific stocks drove the move

### Strategy 4: New Top-10 Momentum
```
🔴 HIGH    Britannia Industries  | 3 AMCs | Max weight: 0.91%
🟡 MEDIUM  Info Edge (India)     | 2 AMCs | Max weight: 3.54%
```
- **🔴 HIGH** = 3+ AMCs independently added this stock this month
- **🟡 MEDIUM** = 2 AMCs added it
- Optimal hold: **60 days**
- Always apply a liquidity filter (avg daily turnover > ₹50Cr) before entering

---

## 🏗️ How This Relates to Agentic AI

| Concept | How it appears in this workflow |
|---|---|
| **Tools** | HTTP Request nodes = tools the agent uses to fetch data |
| **Memory** | `$('node name')` = agent reading previous results |
| **Reasoning** | Code nodes = deterministic reasoning over structured data |
| **Action** | Generating and serving the HTML dashboard |
| **LLM upgrade** | Replace Code nodes with an AI Agent node + Claude/GPT-4 for natural language output |

The difference between this workflow and an LLM agent:
- This workflow uses **hard-coded logic** (rules you define)
- An LLM agent uses **learned reasoning** (the model decides what to do)
- **Best practice**: Use deterministic code for structured financial calculations, LLM for natural language interpretation and explanation

---

## 🚀 Extension Ideas (For Advanced Participants)

1. **Add an AI Agent node** — let GPT-4/Claude explain the signals in plain English
2. **Add email alerts** — when a HIGH conviction signal fires, send an email
3. **Schedule it** — run every month on the 16th (after AMFI disclosure deadline)
4. **Add more funds** — fetch top 5 funds per category instead of 1
5. **Add NAV history** — plot performance of conviction stocks over time
6. **Store in Google Sheets** — save signals monthly to track conviction persistence

---

## 📁 Files Provided

| File | Purpose |
|---|---|
| `mf_alpha_dashboard.json` | Import this into n8n for the complete working dashboard |
| `mf_four_alpha_strategies.json` | Text-only version of the same pipeline (no visualization) |
| `mf_api_quick_test.json` | Tests all 16 API endpoints — use to verify your API key works |
| `mf_template_builder.json` | Pre-fetches and caches data for offline workshop use |

---

## 🔑 API Reference

**Base URL:** `https://app2.mfapis.club/api/v2/public`

**Auth:** Add header `x-api-key: <your key>` to every request

| Endpoint | What it returns |
|---|---|
| `GET /scheme?q=nifty&plan=Direct&option=Growth` | Search mutual funds |
| `GET /scheme/amcs` | List all 50+ AMCs |
| `GET /scheme/filter-options` | All categories, plans, options |
| `GET /scheme/:id` | Full fund details (AUM, expense ratio, returns) |
| `GET /scheme/mf-data/:id` | Morningstar data (holdings, risk metrics) |
| `GET /scheme/:id/related` | Similar funds |
| `GET /scheme/factsheet/:isin` | Factsheet PDF link |
| `GET /nav/:id` | Full NAV history (epoch, value pairs) |
| `GET /amc_portfolio_disclosure/scheme/:isin/holding-changes` | Monthly holdings diff ← **used in this workshop** |
| `GET /amc_portfolio_disclosure/months?isin=X` | Available history months |

---

## ⚠️ Disclaimer

All strategies shown are for **educational purposes only**.
Past performance is not indicative of future results.
This is not investment advice.
Always consult a SEBI-registered investment advisor before making investment decisions.

---

*Built with [n8n](https://n8n.io) + [MfAPIs](https://mfapis.in) · Workshop by MfAPIs*
