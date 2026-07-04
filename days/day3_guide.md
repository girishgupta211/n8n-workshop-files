# Day 3: Portfolio Xray & Analytics Deep Dive

## Learning Objectives

By the end of Day 3 you will be able to:
- Explain the difference between raw portfolio data and analysed Xray output
- Call portfolio analytics APIs including Xray, Health Score, Allocation Analysis, and Factor Exposure
- Schedule a workflow to run automatically every day using the Schedule Trigger
- Store time-series results in a database (Postgres or SQLite)
- Send conditional alerts via Telegram when the health score drops below a threshold
- Run portfolio stress tests against historical market scenarios
- Build a SIP projection calculator workflow

---

## Part 1: Raw Portfolio Data vs Portfolio Xray

### Raw Portfolio Data

When you parse a CAS (Day 2), you get the investor's raw holdings:
- Which schemes they hold
- How many units
- The current NAV and value
- Cost of acquisition and XIRR per scheme

This is useful but limited. It doesn't tell you:
- How much equity vs debt exposure the portfolio has
- Whether large-cap, mid-cap, or small-cap funds dominate
- Whether the portfolio is overly concentrated in one AMC
- How the portfolio would have performed in the 2008 financial crisis
- What factors drive the portfolio's returns (value, growth, momentum, quality)

### Portfolio Xray

Portfolio Xray is a higher-order analysis layer that takes the raw holdings and computes:
- **Asset class allocation**: Equity / Debt / Hybrid / Commodity / REITs %
- **Sub-category breakdown**: Large Cap / Mid Cap / Small Cap / Flexi Cap / ELSS / Gilt / Credit Risk / etc.
- **AMC concentration**: how much of the portfolio is with each AMC
- **Top and bottom performers** by absolute return and XIRR
- **Overlap analysis**: which stocks appear in multiple funds (concentration risk)
- **Risk metrics**: standard deviation, beta, Sharpe ratio relative to benchmark
- **Fund manager exposure**: how much is managed by the same fund manager

### Why Xray Matters for Advisors

An investor might have 12 funds that all appear different but actually hold the same top 20 Nifty stocks. Without Xray, this overlap is invisible. Xray reveals:
- "70% of your portfolio is in the same 50 stocks"
- "3 of your 4 large-cap funds are managed by the same fund manager"
- "Your 'diversified' portfolio is 85% equity — higher than your risk tolerance"

---

## Part 2: Portfolio Analytics API Calls

### getPortfolioXray

Called via the MCP server or the MF Engine API. Returns comprehensive portfolio analysis.

**MCP Tool Call (via AI Agent, Day 5):**
```json
{
  "tool": "getPortfolioXray",
  "parameters": {}
}
```

**Direct HTTP Call:**
```
GET https://app2.mfapis.club/api/v2/portfolio/xray
x-api-key: <your-key>
```

**Response Structure:**
```json
{
  "summary": {
    "total_invested": 500000,
    "current_value": 687000,
    "overall_xirr": 16.2,
    "absolute_returns_pct": 37.4
  },
  "asset_allocation": {
    "equity_pct": 75.2,
    "debt_pct": 18.4,
    "hybrid_pct": 5.1,
    "commodity_pct": 1.3
  },
  "category_breakdown": [
    { "category": "Large Cap", "allocation_pct": 28.4, "value": 195108 },
    { "category": "Mid Cap", "allocation_pct": 22.1, "value": 151827 },
    { "category": "Small Cap", "allocation_pct": 15.8, "value": 108546 },
    { "category": "Flexi Cap", "allocation_pct": 8.9, "value": 61143 },
    { "category": "ELSS", "allocation_pct": 18.4, "value": 126408 }
  ],
  "amc_concentration": [
    { "amc_name": "HDFC Mutual Fund", "allocation_pct": 34.2 },
    { "amc_name": "SBI Mutual Fund", "allocation_pct": 22.8 },
    { "amc_name": "Mirae Asset", "allocation_pct": 18.4 }
  ],
  "top_performers": [
    { "scheme_name": "HDFC Mid Cap Opportunities", "xirr": 22.4 }
  ],
  "bottom_performers": [
    { "scheme_name": "SBI Debt Fund", "xirr": 6.8 }
  ]
}
```

### getPortfolioHealthScore

Returns a single 0–100 score representing overall portfolio health, broken down by component scores.

**Response:**
```json
{
  "overall_score": 72,
  "components": {
    "diversification_score": 65,
    "risk_alignment_score": 78,
    "cost_efficiency_score": 80,
    "returns_quality_score": 70,
    "liquidity_score": 88
  },
  "flags": [
    {
      "type": "concentration_risk",
      "severity": "medium",
      "message": "34% of portfolio in single AMC (HDFC). Consider reducing below 30%."
    },
    {
      "type": "small_cap_overweight",
      "severity": "low",
      "message": "Small cap allocation at 15.8%. Slightly elevated for conservative risk profile."
    }
  ],
  "recommendation_summary": "Portfolio is moderately healthy. Primary concern is AMC concentration in HDFC group."
}
```

### getPortfolioAllocationAnalysis

Provides deeper allocation breakdown including factor exposures, sector concentrations, and comparison to a target allocation.

### getPortfolioFactorExposure

Returns factor loadings for the portfolio:
- **Value**: P/E, P/B based exposure
- **Growth**: revenue growth, earnings growth exposure
- **Momentum**: price momentum exposure (1M, 3M, 12M)
- **Quality**: ROE, ROCE, debt-to-equity exposure
- **Low Volatility**: beta relative to Nifty

### getPortfolioStressTest

Simulates how the portfolio would have performed in historical stress scenarios.

**Parameters:**
- `scenario`: one of `covid_2020`, `gfc_2008`, `demonetization_2016`, `taper_tantrum_2013`, `covid_recovery_2021`

**Response:**
```json
{
  "scenario": "covid_2020",
  "scenario_description": "COVID-19 market crash: Nifty fell 38% from Jan 2020 to March 2020",
  "estimated_portfolio_drawdown_pct": -29.4,
  "estimated_recovery_months": 8,
  "worst_holding": {
    "scheme_name": "Mirae Asset Small Cap",
    "estimated_drawdown_pct": -48.2
  },
  "best_holding": {
    "scheme_name": "SBI Short Duration Fund",
    "estimated_drawdown_pct": -2.1
  },
  "estimated_value_at_trough": 485580,
  "estimated_recovery_value": 720000
}
```

---

## Part 3: Build — Portfolio Health Dashboard

### Workflow Architecture

```
Schedule Trigger (daily at 8:00 AM)
    ↓
HTTP Request: getPortfolioXray
    ↓
HTTP Request: getPortfolioHealthScore
    ↓
HTTP Request: getPortfolioStressTest (covid_2020)
    ↓
HTTP Request: getPortfolioStressTest (gfc_2008) [parallel]
    ↓
Merge: combine stress test results
    ↓
Code Node: compute composite metrics + build DB record
    ↓
Postgres Node: INSERT health record
    ↓
IF Node: health score < 60?
    ├── YES → Telegram alert
    └── NO  → Set Node (log OK)
```

### Step-by-Step Build

#### Step 1: Schedule Trigger

1. Add a **Schedule Trigger** node
2. Trigger Interval: **Days**
3. Days Between Triggers: `1`
4. Trigger at Hour: `8`
5. Trigger at Minute: `0`

This fires every day at 8:00 AM server time.

#### Step 2: HTTP Request — Portfolio Xray

1. Add an **HTTP Request** node named `Get Portfolio Xray`
2. Method: **GET**
3. URL: `https://app2.mfapis.club/api/v2/portfolio/xray`
4. Authentication: **MF Engine API**
5. Headers: you may also need an investor JWT token here
   - Add header: `Authorization: Bearer {{ $env.INVESTOR_JWT }}`

> **Note:** For a partner-level Xray (viewing a client's portfolio), the API accepts `?investor_id=uuid` as a query parameter.

#### Step 3: HTTP Request — Health Score

1. Add an **HTTP Request** node named `Get Health Score`
2. Method: **GET**
3. URL: `https://app2.mfapis.club/api/v2/portfolio/health-score`
4. Authentication: **MF Engine API**

#### Step 4: HTTP Request — Stress Test COVID

1. Add an **HTTP Request** node named `Stress Test COVID 2020`
2. Method: **GET**
3. URL: `https://app2.mfapis.club/api/v2/portfolio/stress-test`
4. Query Params: `scenario=covid_2020`
5. Authentication: **MF Engine API**

#### Step 5: HTTP Request — Stress Test GFC (parallel)

1. Add a second stress test node named `Stress Test GFC 2008`
2. Same as above but `scenario=gfc_2008`
3. Connect this node's input to the Health Score node output (run in parallel with COVID test)

#### Step 6: Merge Node

1. Add a **Merge** node
2. Mode: **Merge By Index**
3. Connect:
   - Input 1: Stress Test COVID 2020
   - Input 2: Stress Test GFC 2008

#### Step 7: Code Node — Compute Composite Metrics

```javascript
const xray = $node['Get Portfolio Xray'].json;
const health = $node['Get Health Score'].json;
const covidStress = $input.item.json; // first item from merge (COVID)
// For GFC, access via items array if needed

const record = {
  recorded_at: new Date().toISOString(),
  investor_id: 'default_investor',
  
  // Summary
  total_invested: xray.summary.total_invested,
  current_value: xray.summary.current_value,
  overall_xirr: xray.summary.overall_xirr,
  
  // Health
  health_score: health.overall_score,
  diversification_score: health.components.diversification_score,
  cost_efficiency_score: health.components.cost_efficiency_score,
  returns_quality_score: health.components.returns_quality_score,
  
  // Allocation
  equity_pct: xray.asset_allocation.equity_pct,
  debt_pct: xray.asset_allocation.debt_pct,
  hybrid_pct: xray.asset_allocation.hybrid_pct,
  
  // Stress
  covid_drawdown_pct: covidStress.estimated_portfolio_drawdown_pct,
  covid_recovery_months: covidStress.estimated_recovery_months,
  
  // Flags
  active_flags: health.flags.length,
  high_severity_flags: health.flags.filter(f => f.severity === 'high').length
};

return [{ json: record }];
```

#### Step 8: Postgres Node — INSERT Health Record

1. Add a **Postgres** node (or **SQLite** if you prefer)
2. Operation: **Execute Query**
3. Query:
```sql
INSERT INTO portfolio_health_daily (
  recorded_at, investor_id, total_invested, current_value, overall_xirr,
  health_score, diversification_score, cost_efficiency_score, returns_quality_score,
  equity_pct, debt_pct, hybrid_pct, covid_drawdown_pct, covid_recovery_months,
  active_flags, high_severity_flags
) VALUES (
  '{{ $json.recorded_at }}',
  '{{ $json.investor_id }}',
  {{ $json.total_invested }},
  {{ $json.current_value }},
  {{ $json.overall_xirr }},
  {{ $json.health_score }},
  {{ $json.diversification_score }},
  {{ $json.cost_efficiency_score }},
  {{ $json.returns_quality_score }},
  {{ $json.equity_pct }},
  {{ $json.debt_pct }},
  {{ $json.hybrid_pct }},
  {{ $json.covid_drawdown_pct }},
  {{ $json.covid_recovery_months }},
  {{ $json.active_flags }},
  {{ $json.high_severity_flags }}
);
```

**Database Setup (run once):**
```sql
CREATE TABLE portfolio_health_daily (
  id SERIAL PRIMARY KEY,
  recorded_at TIMESTAMP NOT NULL,
  investor_id VARCHAR(100) NOT NULL,
  total_invested DECIMAL(15,2),
  current_value DECIMAL(15,2),
  overall_xirr DECIMAL(6,2),
  health_score INTEGER,
  diversification_score INTEGER,
  cost_efficiency_score INTEGER,
  returns_quality_score INTEGER,
  equity_pct DECIMAL(5,2),
  debt_pct DECIMAL(5,2),
  hybrid_pct DECIMAL(5,2),
  covid_drawdown_pct DECIMAL(6,2),
  covid_recovery_months INTEGER,
  active_flags INTEGER,
  high_severity_flags INTEGER,
  created_at TIMESTAMP DEFAULT NOW()
);
```

#### Step 9: IF Node — Health Score Alert Check

1. Add an **IF** node named `Health Score Alert?`
2. Conditions (AND):
   - `{{ $json.health_score }}` is less than `60`

#### Step 10: Telegram Alert (True Branch)

1. Add a **Telegram** node (requires Telegram Bot Token credential)
2. Operation: **Send Message**
3. Chat ID: your advisor Telegram chat ID
4. Message:
```
⚠️ Portfolio Health Alert

Investor: {{ $json.investor_id }}
Health Score: {{ $json.health_score }}/100 (BELOW THRESHOLD)

Portfolio Value: ₹{{ $json.current_value.toLocaleString() }}
XIRR: {{ $json.overall_xirr }}%

Equity Allocation: {{ $json.equity_pct }}%
Active Flags: {{ $json.active_flags }} ({{ $json.high_severity_flags }} high severity)

COVID Stress Drawdown: {{ $json.covid_drawdown_pct }}%

Please review the portfolio dashboard immediately.
```

#### Step 11: Set Node (False Branch — OK Status)

```javascript
{
  status: "OK",
  health_score: "{{ $json.health_score }}",
  message: "Health score above threshold. No alert sent.",
  checked_at: "{{ $now }}"
}
```

---

## Part 4: SIP Projection Calculator Workflow

A SIP (Systematic Investment Plan) projection calculator helps advisors show clients how their wealth grows with regular monthly investments.

### Standalone SIP Projection Workflow

```
Webhook (POST: {sip_amount, years, expected_return_pct, fund_name})
    ↓
HTTP Request: getSipProjection (via MCP or direct)
    ↓
Code Node: build projection table
    ↓
Set Node: format response
    ↓
Respond to Webhook
```

### Code Node — SIP Math

If the API doesn't have a direct SIP projection endpoint, compute it yourself:

```javascript
const { sip_amount, years, expected_return_pct } = $input.first().json.body;

const monthlyRate = expected_return_pct / 100 / 12;
const months = years * 12;

// Future value of SIP (annuity formula)
const futureValue = sip_amount * ((Math.pow(1 + monthlyRate, months) - 1) / monthlyRate) * (1 + monthlyRate);
const totalInvested = sip_amount * months;
const wealthGain = futureValue - totalInvested;

// Year-by-year projection
const yearlyProjection = [];
for (let y = 1; y <= years; y++) {
  const m = y * 12;
  const fv = sip_amount * ((Math.pow(1 + monthlyRate, m) - 1) / monthlyRate) * (1 + monthlyRate);
  const invested = sip_amount * m;
  yearlyProjection.push({
    year: y,
    invested: Math.round(invested),
    portfolio_value: Math.round(fv),
    gain: Math.round(fv - invested)
  });
}

return [{
  json: {
    sip_amount,
    years,
    expected_return_pct,
    total_invested: Math.round(totalInvested),
    projected_value: Math.round(futureValue),
    wealth_gain: Math.round(wealthGain),
    absolute_return_pct: Math.round((wealthGain / totalInvested) * 100 * 10) / 10,
    yearly_projection: yearlyProjection
  }
}];
```

### Testing SIP Projection

```bash
curl -X POST http://localhost:5678/webhook-test/sip-projection \
  -H "Content-Type: application/json" \
  -d '{"sip_amount": 10000, "years": 15, "expected_return_pct": 14, "fund_name": "HDFC Mid Cap"}'
```

Expected output for ₹10,000/month SIP at 14% for 15 years:
- Total Invested: ₹18,00,000
- Projected Value: ₹60,38,000
- Wealth Gain: ₹42,38,000 (235% absolute return)

---

## Part 5: Querying Historical Health Data

Once you have been storing daily health records, you can query trends:

### Add a Second Workflow — Health Trend Query

```
Webhook (GET /health-trend?investor_id=xxx&days=30)
    ↓
Postgres Node: SELECT last 30 days
    ↓
Code Node: compute trend (improving/declining/stable)
    ↓
Respond to Webhook
```

**Postgres Query:**
```sql
SELECT recorded_at, health_score, equity_pct, overall_xirr, active_flags
FROM portfolio_health_daily
WHERE investor_id = '{{ $json.query.investor_id }}'
  AND recorded_at >= NOW() - INTERVAL '{{ $json.query.days || 30 }} days'
ORDER BY recorded_at ASC;
```

**Code Node — Trend Analysis:**
```javascript
const records = $input.all().map(i => i.json);

if (records.length < 2) {
  return [{ json: { trend: 'insufficient_data', records } }];
}

const first = records[0].health_score;
const last = records[records.length - 1].health_score;
const change = last - first;
const trend = change > 3 ? 'improving' : change < -3 ? 'declining' : 'stable';

const avgScore = records.reduce((s, r) => s + r.health_score, 0) / records.length;
const minScore = Math.min(...records.map(r => r.health_score));
const maxScore = Math.max(...records.map(r => r.health_score));

return [{
  json: {
    trend,
    change_over_period: change,
    average_score: Math.round(avgScore * 10) / 10,
    min_score: minScore,
    max_score: maxScore,
    days_analyzed: records.length,
    records
  }
}];
```

---

## Part 6: Exercises

### Exercise 1: Multi-Scenario Stress Test

Extend the workflow to run stress tests for all 5 scenarios in parallel:
- `covid_2020`
- `gfc_2008`
- `demonetization_2016`
- `taper_tantrum_2013`
- `covid_recovery_2021`

Use multiple parallel HTTP Request nodes, merge them all, and store the worst-case scenario drawdown in the health record.

### Exercise 2: Alert Threshold Configuration

Make the alert threshold configurable without editing the workflow:
1. Store the threshold in an N8N environment variable (`HEALTH_SCORE_ALERT_THRESHOLD`)
2. Reference it in the IF node: `{{ $env.HEALTH_SCORE_ALERT_THRESHOLD }}`
3. Default to 60 if not set: `{{ $env.HEALTH_SCORE_ALERT_THRESHOLD || 60 }}`

### Exercise 3: Weekly Summary Email

Add a second schedule trigger that fires every Monday at 9:00 AM. This trigger fetches the last 7 days of health records from Postgres and emails a weekly summary to the advisor showing:
- Average health score for the week
- Number of alerts triggered
- Best and worst day by health score
- Change in portfolio value over the week

---

## Day 3 Summary

You learned how to:
- Differentiate raw portfolio data from Xray analysis
- Call portfolio health, Xray, allocation, and stress test APIs
- Schedule workflows with the Schedule Trigger node
- Store time-series data in Postgres with SQL INSERT
- Send conditional Telegram alerts based on computed thresholds
- Build a SIP projection calculator
- Query and analyse trend data from stored records

**Day 3 workflow to import:** `workflows/day3_portfolio_xray.json`

Tomorrow in Day 4 you will combine MF portfolio disclosure data (which stocks funds are buying) with the Stoic stock screener to identify institutional alpha signals.
