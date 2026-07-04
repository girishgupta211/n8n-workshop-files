# Day 4: Stoic Stocks Data + MF Data Integration

## Learning Objectives

By the end of Day 4 you will be able to:
- Explain the Stoic systematic screening methodology
- Interpret RSI, MACD, Bollinger Bands, and SMA crossover signals
- Understand AMC Portfolio Disclosures and how to track institutional buying
- Fetch and process both stocks screener and fund disclosure data in N8N
- Perform a cross-dataset join in a Code node (SQL-style JOIN in JavaScript)
- Build an Institutional Alpha Scanner that identifies oversold stocks held by multiple funds
- Build a Fund-vs-Stock correlation pipeline to find which funds hold a given stock

---

## Part 1: The Stoic Stock Screener

### What is STOIC?

STOIC stands for **Systematic, Technical, Objective, Informed, Consistent**. It is a disciplined, rules-based approach to stock analysis that combines:

- **Technical indicators** (RSI, MACD, Bollinger Bands, SMA) for entry/exit timing
- **Fundamental filters** (PE, PB, ROE, ROCE, D/E) for quality screening
- **Volume and delivery metrics** for institutional activity signals
- **Price structure** (52-week high/low proximity) for momentum context

The Stoic screener endpoint returns a pre-computed universe of stocks with all indicators calculated, so you do not need to write indicator logic yourself.

### Stoic Screener API

```
GET https://app2.mfapis.club/api/v2/stocks?type=equity_screener&limit=50
x-api-key: <your-api-key>
```

**Response structure per stock:**
```json
{
  "symbol": "RELIANCE",
  "name": "Reliance Industries Limited",
  "exchange": "NSE",
  "sector": "Energy",
  "industry": "Oil & Gas Refining",
  "last_price": 2847.50,
  "market_cap_cr": 1924000,
  
  "technical": {
    "rsi_14": 38.2,
    "macd": 12.4,
    "macd_signal": 18.9,
    "macd_histogram": -6.5,
    "bb_upper": 3100.0,
    "bb_lower": 2680.0,
    "bb_middle": 2890.0,
    "sma_50": 2910.0,
    "sma_200": 2750.0,
    "atr_14": 54.2,
    "volume_ratio_20d": 1.42,
    "high_52w": 3220.0,
    "low_52w": 2180.0,
    "pct_from_52w_high": -11.6,
    "pct_from_52w_low": 30.6,
    "delivery_ratio_20d": 0.52
  },
  
  "fundamental": {
    "pe": 22.4,
    "pb": 2.1,
    "roe": 9.8,
    "roce": 12.1,
    "de_ratio": 0.42,
    "div_yield_pct": 0.35
  }
}
```

### Reading the Indicators

#### RSI (Relative Strength Index, 14-period)
- Range: 0–100
- Below 30: **Oversold** — potential buy signal (stock may be undervalued technically)
- 30–70: **Neutral** — normal range
- Above 70: **Overbought** — potential sell signal or consolidation ahead
- The Stoic screener flags RSI < 40 as "mildly oversold" (a more conservative threshold)

#### MACD (Moving Average Convergence Divergence)
- MACD Line = 12-period EMA minus 26-period EMA
- Signal Line = 9-period EMA of MACD line
- Histogram = MACD minus Signal
- **Bullish signal**: MACD crosses above Signal (histogram turns positive)
- **Bearish signal**: MACD crosses below Signal (histogram turns negative)
- A stock with MACD < 0 and MACD < Signal is in a bearish momentum phase

#### Bollinger Bands (20-period, 2 standard deviations)
- When price touches or breaks **BB Lower**: potential oversold / mean-reversion candidate
- When price touches or breaks **BB Upper**: potential overbought
- `bb_lower` gives you the lower band value — compare to `last_price`

#### SMA Crossovers
- **Golden Cross**: SMA 50 crosses above SMA 200 — long-term bullish trend shift
- **Death Cross**: SMA 50 crosses below SMA 200 — long-term bearish trend shift
- Current state: if `sma_50 > sma_200`, the stock is in a golden-cross regime

#### Volume Ratio (20-day)
- `volume_ratio_20d = current_volume / 20d_average_volume`
- Ratio > 1.5: significantly above-average volume (institutional activity or news)
- Combined with an oversold RSI: high volume + oversold = potential institutional accumulation

#### Delivery Ratio (20-day)
- Delivery % = what percentage of traded volume was actually delivered (not day-traded)
- Higher delivery ratio suggests genuine investor conviction vs speculative trading

---

## Part 2: AMC Portfolio Disclosures

### What Are Portfolio Disclosures?

SEBI requires every mutual fund to disclose its full equity holdings every month. These disclosures show:
- Which stocks the fund holds
- How many shares they hold
- What percentage of the fund's AUM is in each stock

This data is goldmine for retail investors because:
1. You can see what the smartest fund managers are buying and selling
2. Month-over-month changes reveal new positions (accumulation) and exits (distribution)
3. Stocks held by multiple funds have "institutional conviction" — multiple research teams agree the stock is good

### AMC Portfolio Disclosure API

```
GET https://app2.mfapis.club/api/v2/amc_portfolio_disclosure?limit=100&month=2025-06
x-api-key: <your-api-key>
```

**Parameters:**
- `month`: format YYYY-MM (e.g., 2025-06). Omit for latest available month
- `limit`: max number of records to return
- `amc_name`: filter to a specific AMC (optional)
- `scheme_id`: filter to a specific fund (optional)

**Response:**
```json
[
  {
    "fund_name": "HDFC Mid Cap Opportunities Fund",
    "amc_name": "HDFC Mutual Fund",
    "scheme_id": "uuid-hdfc-mid",
    "month": "2025-06",
    "portfolio": [
      {
        "stock_symbol": "TATAELXSI",
        "stock_name": "Tata Elxsi Limited",
        "isin": "INE670A01012",
        "sector": "Technology",
        "market_value_cr": 842.5,
        "pct_of_nav": 1.23,
        "shares_held": 287500,
        "month_change_pct": 0.08
      },
      {
        "stock_symbol": "CAMS",
        "stock_name": "Computer Age Management Services",
        "isin": "INE596I01012",
        "sector": "Financial Services",
        "market_value_cr": 623.1,
        "pct_of_nav": 0.91,
        "shares_held": 180000,
        "month_change_pct": -0.12
      }
    ]
  }
]
```

---

## Part 3: Build — Institutional Alpha Scanner

The Institutional Alpha Scanner finds stocks that meet two conditions simultaneously:
1. **Institutional conviction**: held by 3 or more distinct mutual funds
2. **Technical opportunity**: RSI < 40 (mildly oversold)

The logic: if smart money (multiple fund managers) has done the research and holds the stock, and the stock is temporarily oversold, it may be a buying opportunity.

### Workflow Architecture

```
Schedule Trigger (weekly, Monday 7:00 AM)
    ↓
HTTP Request: GET /amc_portfolio_disclosure (current month)
    ↓
HTTP Request: GET /stocks?type=equity_screener (parallel)
    ↓
Merge: wait for both
    ↓
Code Node: build stock-to-fund index
    ↓
Code Node: join — stocks in 3+ funds AND RSI < 40
    ↓
Set Node: format watchlist output
    ↓
Respond to Webhook (or store to Airtable)
```

### Step-by-Step Build

#### Step 1: Schedule Trigger

1. Add a **Schedule Trigger** node
2. Interval: **Weeks**
3. Day of Week: **Monday**
4. Hour: `7`
5. Minute: `0`

#### Step 2: Fetch Portfolio Disclosures

1. Add an **HTTP Request** node named `Fetch Fund Holdings`
2. Method: **GET**
3. URL: `https://app2.mfapis.club/api/v2/amc_portfolio_disclosure`
4. Query Params:
   - `limit`: `200`
   - `month`: `{{ new Date().toISOString().slice(0, 7) }}`
5. Authentication: **MF Engine API**

#### Step 3: Fetch Stoic Screener (parallel)

1. Add an **HTTP Request** node named `Fetch Stoic Screener`
2. Connect input to the **Schedule Trigger** (same trigger, parallel path)
3. Method: **GET**
4. URL: `https://app2.mfapis.club/api/v2/stocks`
5. Query Params:
   - `type`: `equity_screener`
   - `limit`: `200`
6. Authentication: **MF Engine API**

#### Step 4: Merge — Wait for Both

1. Add a **Merge** node
2. Mode: **Wait** (waits for both inputs to have data)
3. Connect:
   - Input 1: Fetch Fund Holdings
   - Input 2: Fetch Stoic Screener

#### Step 5: Code Node — Build Stock-to-Fund Index

```javascript
// $input now has items from both paths after the Merge
// We need to access both nodes separately
const disclosureData = $node['Fetch Fund Holdings'].json;
const screenerData = $node['Fetch Stoic Screener'].json;

// Build index: symbol -> list of funds that hold it
const stockToFunds = {};

const funds = Array.isArray(disclosureData) ? disclosureData : [disclosureData];

funds.forEach(fund => {
  const holdings = fund.portfolio || [];
  holdings.forEach(holding => {
    const sym = holding.stock_symbol;
    if (!stockToFunds[sym]) {
      stockToFunds[sym] = {
        symbol: sym,
        stock_name: holding.stock_name,
        sector: holding.sector,
        funds: []
      };
    }
    stockToFunds[sym].funds.push({
      fund_name: fund.fund_name,
      amc_name: fund.amc_name,
      pct_of_nav: holding.pct_of_nav,
      market_value_cr: holding.market_value_cr,
      month_change_pct: holding.month_change_pct
    });
  });
});

// Build index: symbol -> screener data
const screenerIndex = {};
const stocks = Array.isArray(screenerData) ? screenerData : [screenerData];
stocks.forEach(s => {
  screenerIndex[s.symbol] = s;
});

return [{
  json: {
    stock_to_funds: stockToFunds,
    screener_index: screenerIndex
  }
}];
```

#### Step 6: Code Node — Apply Alpha Filter

```javascript
const { stock_to_funds, screener_index } = $input.first().json;

const MIN_FUNDS = 3;
const MAX_RSI = 40;

const alphaSignals = [];

Object.entries(stock_to_funds).forEach(([symbol, data]) => {
  if (data.funds.length < MIN_FUNDS) return; // Not enough institutional conviction
  
  const screener = screener_index[symbol];
  if (!screener) return; // Not in screener universe
  
  const rsi = screener.technical?.rsi_14;
  if (rsi === undefined || rsi >= MAX_RSI) return; // Not oversold
  
  // Compute additional signals
  const macd = screener.technical.macd;
  const macdSignal = screener.technical.macd_signal;
  const isBearishMomentum = macd < macdSignal;
  
  const price = screener.last_price;
  const bbLower = screener.technical.bb_lower;
  const nearBBLower = price <= bbLower * 1.02; // within 2% of lower band
  
  const sma50 = screener.technical.sma_50;
  const sma200 = screener.technical.sma_200;
  const aboveSma200 = price > sma200; // long-term uptrend maintained
  
  // Compute institutional net conviction
  const avgPctOfNav = data.funds.reduce((s, f) => s + f.pct_of_nav, 0) / data.funds.length;
  const fundsBuying = data.funds.filter(f => f.month_change_pct > 0).length;
  const fundsSelling = data.funds.filter(f => f.month_change_pct < 0).length;
  
  // Score the signal (higher = stronger)
  let signalScore = 0;
  signalScore += (MAX_RSI - rsi); // lower RSI = higher score
  signalScore += data.funds.length * 5; // more funds = stronger
  if (nearBBLower) signalScore += 10;
  if (aboveSma200) signalScore += 8; // above long-term MA is constructive
  if (fundsBuying > fundsSelling) signalScore += 5;
  
  alphaSignals.push({
    symbol,
    name: data.stock_name,
    sector: data.sector,
    last_price: price,
    
    // Technical
    rsi_14: rsi,
    macd_histogram: screener.technical.macd_histogram,
    near_bb_lower: nearBBLower,
    above_sma_200: aboveSma200,
    sma_50: sma50,
    sma_200: sma200,
    volume_ratio: screener.technical.volume_ratio_20d,
    pct_from_52w_high: screener.technical.pct_from_52w_high,
    
    // Fundamental
    pe: screener.fundamental?.pe,
    roe: screener.fundamental?.roe,
    
    // Institutional
    fund_count: data.funds.length,
    funds_holding: data.funds.map(f => f.fund_name),
    avg_pct_of_nav: Math.round(avgPctOfNav * 100) / 100,
    funds_adding: fundsBuying,
    funds_trimming: fundsSelling,
    
    // Signal
    signal_score: Math.round(signalScore * 10) / 10,
    signal_strength: signalScore > 50 ? 'STRONG' : signalScore > 30 ? 'MODERATE' : 'WEAK'
  });
});

// Sort by signal score descending
alphaSignals.sort((a, b) => b.signal_score - a.signal_score);

return [{
  json: {
    generated_at: new Date().toISOString(),
    filter_criteria: {
      min_funds: MIN_FUNDS,
      max_rsi: MAX_RSI
    },
    signal_count: alphaSignals.length,
    signals: alphaSignals
  }
}];
```

#### Step 7: Set Node — Format Watchlist Output

```javascript
// Mode: Only Specified Fields
{
  watchlist_date: "{{ $json.generated_at.slice(0, 10) }}",
  total_signals: "{{ $json.signal_count }}",
  top_10_signals: "{{ $json.signals.slice(0, 10) }}",
  filter_used: "{{ JSON.stringify($json.filter_criteria) }}"
}
```

---

## Part 4: Build — Fund-vs-Stock Correlation Pipeline

Given a stock symbol, find all mutual funds that hold it and their conviction level.

### Workflow Architecture

```
Webhook (GET /fund-lookup?stock=TATAELXSI)
    ↓
HTTP Request: GET /amc_portfolio_disclosure?limit=200
    ↓
Code Node: filter to stocks matching query
    ↓
HTTP Request: GET /stocks/fundamentals?symbol={stock} (parallel)
    ↓
Merge
    ↓
Code Node: build fund conviction table
    ↓
Respond to Webhook
```

### Code Node — Filter and Build Table

```javascript
const stockQuery = $node['Webhook'].json.query.stock.toUpperCase();
const disclosures = $node['Fetch Fund Holdings'].json;
const fundamentals = $node['Stock Fundamentals'].json;

const matchingFunds = [];

const funds = Array.isArray(disclosures) ? disclosures : [disclosures];

funds.forEach(fund => {
  const holdings = fund.portfolio || [];
  const holding = holdings.find(h => h.stock_symbol === stockQuery);
  if (holding) {
    matchingFunds.push({
      fund_name: fund.fund_name,
      amc_name: fund.amc_name,
      scheme_id: fund.scheme_id,
      pct_of_nav: holding.pct_of_nav,
      market_value_cr: holding.market_value_cr,
      shares_held: holding.shares_held,
      month_change_pct: holding.month_change_pct,
      position_direction: holding.month_change_pct > 0 ? 'INCREASING' : holding.month_change_pct < 0 ? 'DECREASING' : 'UNCHANGED'
    });
  }
});

// Sort by pct_of_nav (conviction level)
matchingFunds.sort((a, b) => b.pct_of_nav - a.pct_of_nav);

const totalMarketValueCr = matchingFunds.reduce((s, f) => s + f.market_value_cr, 0);
const fundsAdding = matchingFunds.filter(f => f.month_change_pct > 0).length;

return [{
  json: {
    stock_symbol: stockQuery,
    fundamentals: {
      pe: fundamentals.pe,
      pb: fundamentals.pb,
      roe: fundamentals.roe,
      market_cap_cr: fundamentals.market_cap_cr,
      div_yield_pct: fundamentals.div_yield_pct
    },
    institutional_summary: {
      total_funds_holding: matchingFunds.length,
      total_institutional_value_cr: Math.round(totalMarketValueCr * 100) / 100,
      funds_increasing_position: fundsAdding,
      funds_decreasing_position: matchingFunds.length - fundsAdding,
      net_institutional_sentiment: fundsAdding > matchingFunds.length / 2 ? 'BULLISH' : 'BEARISH'
    },
    fund_positions: matchingFunds
  }
}];
```

### Testing Fund Lookup

```bash
curl "http://localhost:5678/webhook-test/fund-lookup?stock=TATAELXSI"
```

Expected: list of funds holding TATAELXSI, sorted by conviction (% of NAV), with net institutional sentiment.

---

## Part 5: MF Data Deep Dive — Other Endpoints

### AMC Factsheet API

```
GET /api/v2/amc_factsheet?amc_name=HDFC&limit=20
```

Returns factsheet data for funds under the specified AMC:
- Fund manager details
- AUM over time
- Risk metrics (standard deviation, beta, Sharpe)
- Holdings count and turnover ratio

Useful for: comparing AMCs, tracking AUM flows, identifying manager changes.

### MF Universe Query

```
GET /api/v2/mf_query?category=Mid+Cap&limit=20
```

Filter the MF universe by:
- `category`: Large Cap, Mid Cap, Small Cap, Flexi Cap, ELSS, Gilt, Liquid, etc.
- `amc_name`: filter to specific AMC
- `min_aum_cr`: minimum AUM filter
- `max_expense_ratio`: expense ratio cap

Use this to build fund shortlists for comparison.

### F&O Screener

```
GET /api/v2/stocks?type=fno_screener&limit=50
```

Returns F&O stocks with:
- `max_pain`: the strike price at which option sellers have maximum profit at expiry
- `pcr_oi`: Put-Call Ratio by Open Interest (PCR OI > 1.2 = bearish sentiment)
- `pcr_vol`: Put-Call Ratio by Volume
- `futures_premium`: cost-of-carry premium
- `iv_percentile`: implied volatility percentile (high = expensive options)
- `rollover_pct`: percentage of positions rolled to next series
- `oi_change_pct`: open interest change % (OI increasing with price = bullish)

Useful for: options strategy planning, identifying stocks with high/low volatility regimes.

---

## Part 6: Exercises

### Exercise 1: Sector Concentration in Disclosures

Modify the Alpha Scanner to also compute sector-level concentration:
1. For each sector, count how many of the disclosed funds are holding stocks in that sector
2. Identify the top 3 sectors with the highest institutional coverage
3. Add a `sector_bias` field to the output

### Exercise 2: Month-over-Month Change Tracker

Add a second workflow that:
1. Fetches disclosures for the current month AND the previous month
2. Compares the two datasets to find:
   - New entries: stocks that appeared in a fund's portfolio this month but not last month
   - Exits: stocks that disappeared from a fund's portfolio
   - Conviction increases: stocks where `pct_of_nav` increased by more than 0.2%

### Exercise 3: Combined Technical Filter

Extend the Alpha Scanner with additional technical filters. Create a "conviction tier" system:
- **Tier 1 (Highest)**: RSI < 35 AND price < BB Lower AND Volume Ratio > 1.5 AND 3+ funds
- **Tier 2 (High)**: RSI < 40 AND price < BB Lower AND 3+ funds
- **Tier 3 (Moderate)**: RSI < 40 AND 3+ funds
- **Tier 4 (Watch)**: RSI < 40 AND 2 funds

### Exercise 4: Stock OHLCV Chart Data

For each stock in the watchlist, fetch 30 days of OHLCV data and compute:
- 30-day return
- Volatility (standard deviation of daily returns)
- Highest volume day in the period

```bash
GET /api/v2/stocks/ohlcv?symbol=RELIANCE&exchange=NSE&period=30d
```

---

## Day 4 Summary

You learned how to:
- Interpret the Stoic screener's RSI, MACD, Bollinger Band, and SMA indicators
- Fetch AMC portfolio disclosures and understand what they reveal about institutional activity
- Perform a cross-dataset join in JavaScript to combine two independent data sources
- Build an Institutional Alpha Scanner with signal scoring
- Build a Fund-vs-Stock reverse lookup pipeline
- Use additional MF API endpoints (factsheets, MF universe query, F&O screener)

**Day 4 workflow to import:** `workflows/day4_alpha_scanner.json`

Tomorrow in Day 5 you will integrate MCP servers into N8N AI Agent nodes, enabling natural language portfolio advisory queries powered by structured tool calls.
