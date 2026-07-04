# Day 1: N8N Fundamentals + MF APIs Introduction

## Learning Objectives

By the end of Day 1 you will be able to:
- Explain the core N8N node types and how data flows between them
- Configure Header Auth credentials for the MF Engine API
- Build a working HTTP GET request to search for mutual fund schemes
- Chain multiple API calls in a single workflow
- Display formatted output using the Set node
- Trigger workflows via webhook and test them with curl or Postman

---

## Part 1: N8N Core Concepts

### What is N8N?

N8N is an open-source, node-based workflow automation platform. Think of it as a visual programming environment where each box (node) does one job, and you wire boxes together to build pipelines. Unlike Zapier or Make (which are primarily SaaS), N8N is self-hostable and gives you direct control over data, credentials, and execution logic.

### The N8N Data Model

Every node in N8N receives and emits **items**. An item is a JSON object. When a node runs, it processes an array of items and outputs a new array of items. This is the single most important concept:

```
[Node A] → outputs [{a:1}, {a:2}, {a:3}]
[Node B] → receives [{a:1}, {a:2}, {a:3}], outputs [{b:10}, {b:20}, {b:30}]
```

Each node processes all items it receives. Most nodes process items one at a time (per-item execution). The Code node lets you process the entire array at once.

### Expressions and the `$` Syntax

Inside any node's parameter field you can use expressions wrapped in `{{ }}`. Expressions give you access to:

- `{{ $json.fieldName }}` — a field from the current item's JSON
- `{{ $node['Node Name'].json.fieldName }}` — data from a specific upstream node
- `{{ $runIndex }}` — current iteration index
- `{{ $now }}` — current timestamp
- `{{ $env.MY_VAR }}` — environment variable

Example: if the HTTP Response node returns `{ "scheme_name": "HDFC Mid Cap" }`, then in a downstream Set node you can write `{{ $json.scheme_name }}` to reference it.

---

## Part 2: Core Nodes You Will Use Today

### Webhook Node (`n8n-nodes-base.webhook`)

The Webhook node creates an HTTP endpoint that triggers your workflow. It supports GET and POST. When triggered, the incoming request body/query parameters become the first item in the workflow.

**Key settings:**
- HTTP Method: GET or POST
- Path: the URL path suffix (e.g., `/scheme-search`)
- Response Mode: `Last Node` (respond with the final node's output) or `Respond Immediately`

**Test URL vs Production URL:** N8N gives you two URLs — a test URL (only active when you click "Listen for test event") and a production URL (always active when the workflow is active).

### HTTP Request Node (`n8n-nodes-base.httpRequest`)

The workhorse node for calling external APIs. Supports all HTTP methods, query parameters, body types (JSON, form data, multipart), and authentication.

**Key settings:**
- Method: GET / POST / PUT / DELETE
- URL: the endpoint URL (can use expressions)
- Authentication: link to a saved credential
- Query Parameters: key-value pairs appended to the URL
- Send Body: for POST/PUT, choose JSON or Form Data

**Result:** the API response body is parsed and available as `$json` in downstream nodes.

### Set Node (`n8n-nodes-base.set`)

The Set node reshapes data. Use it to:
- Pick only the fields you care about
- Rename fields
- Add computed fields using expressions
- Build the output structure you want to return

**Mode options:**
- **Keep All Fields**: adds new fields without removing existing ones
- **Only Specified Fields**: starts fresh — only the fields you define appear in output

### IF Node (`n8n-nodes-base.if`)

Branches the workflow based on a condition. Outputs two streams: `true` branch and `false` branch. Conditions can check string equality, number comparisons, regex matches, or whether a field exists.

### Code Node (`n8n-nodes-base.code`)

Runs JavaScript code. Two modes:
- **Run Once for All Items**: receives `$input.all()` — the full array. Return an array of items.
- **Run Once per Item**: receives `$input.item` — one item at a time. Return a single item.

Use the Code node when you need complex transformations that expressions alone can't handle.

### Merge Node (`n8n-nodes-base.merge`)

Combines data from two branches. Merge modes:
- **Append**: concatenate both arrays
- **Merge By Index**: zip items by position
- **Merge By Key**: join on a shared field (like a SQL join)
- **Wait**: wait for both branches to complete, then merge

### Respond to Webhook Node (`n8n-nodes-base.respondToWebhook`)

Sends an HTTP response back to the webhook caller. Use this when Webhook Response Mode is set to `Using Respond to Webhook Node`. Lets you set the response body, status code, and headers.

---

## Part 3: Setting Up Credentials

### Adding the MF Engine API Credential

1. In N8N, go to **Settings** (gear icon) → **Credentials**
2. Click **Add Credential**
3. Search for **Header Auth** and select it
4. Fill in:
   - **Credential Name**: `MF Engine API`
   - **Name** (header name): `x-api-key`
   - **Value**: your API key (e.g., `mfe_live_abc123...`)
5. Click **Save**

The credential is now available to all HTTP Request nodes in your instance.

### Using the Credential in HTTP Request Nodes

1. Open an HTTP Request node
2. Under **Authentication**, select **Generic Credential Type**
3. In the dropdown, select **Header Auth**
4. In the **Credential** dropdown, select **MF Engine API**
5. The node will now send `x-api-key: <your-key>` with every request

---

## Part 4: Build — Scheme Explorer Workflow

The Scheme Explorer takes a fund name as input, searches for matching schemes, fetches the current NAV for the top result, and returns a formatted fund profile.

### Workflow Architecture

```
Webhook (GET /scheme-search?name=hdfc)
    ↓
HTTP Request: GET /api/v2/scheme?search={name}&limit=5
    ↓
Set Node: extract top result
    ↓
HTTP Request: GET /api/v2/nav?scheme_id={scheme_id}&limit=1
    ↓
HTTP Request: GET /api/v2/scheme/{scheme_id}
    ↓
Merge: combine NAV + scheme details
    ↓
Set Node: build fund profile response
    ↓
Respond to Webhook
```

### Step-by-Step Build

#### Step 1: Add Webhook Node

1. Create a new workflow
2. Add a **Webhook** node
3. Set HTTP Method: **GET**
4. Set Path: `scheme-search`
5. Set Response Mode: `Using 'Respond to Webhook' Node`
6. Note the test URL: `http://localhost:5678/webhook-test/scheme-search`

#### Step 2: Add HTTP Request — Search Schemes

1. Add an **HTTP Request** node connected to the Webhook
2. Name it `Search Schemes`
3. Method: **GET**
4. URL: `https://app2.mfapis.club/api/v2/scheme`
5. Under **Query Parameters**, add:
   - Key: `search`, Value: `{{ $json.query.name }}`
   - Key: `limit`, Value: `5`
6. Authentication: select **MF Engine API** credential

**What you get back:** an array of scheme objects, each with:
```json
{
  "scheme_id": "uuid-here",
  "scheme_name": "HDFC Mid Cap Opportunities Fund",
  "amc_name": "HDFC Mutual Fund",
  "category": "Mid Cap",
  "sub_category": "Mid Cap Fund",
  "expense_ratio": 1.62,
  "aum_cr": 68432.50
}
```

#### Step 3: Add Set Node — Extract Top Result

1. Add a **Set** node named `Extract Top Result`
2. Mode: **Only Specified Fields**
3. Add these fields:
   - `scheme_id`: `{{ $json[0].scheme_id }}`
   - `scheme_name`: `{{ $json[0].scheme_name }}`
   - `amc_name`: `{{ $json[0].amc_name }}`
   - `category`: `{{ $json[0].category }}`
   - `expense_ratio`: `{{ $json[0].expense_ratio }}`

> **Note:** `$json` here is the response body from the search API. If the API returns items as an array directly, use `$json[0]`. If wrapped in a `data` key, use `$json.data[0]`.

#### Step 4: Add HTTP Request — Fetch Latest NAV

1. Add a second **HTTP Request** node named `Fetch Latest NAV`
2. Method: **GET**
3. URL: `https://app2.mfapis.club/api/v2/nav`
4. Query Parameters:
   - Key: `scheme_id`, Value: `{{ $json.scheme_id }}`
   - Key: `limit`, Value: `1`
5. Authentication: **MF Engine API**

**What you get back:**
```json
[{
  "date": "2025-07-01",
  "nav": 98.432,
  "scheme_id": "uuid-here"
}]
```

#### Step 5: Add HTTP Request — Fetch Scheme Details

1. Add a third **HTTP Request** node named `Fetch Scheme Details`
2. Connect it to the **Extract Top Result** Set node (parallel to NAV fetch)
3. Method: **GET**
4. URL: `https://app2.mfapis.club/api/v2/scheme/{{ $json.scheme_id }}`
5. Authentication: **MF Engine API**

**What you get back:**
```json
{
  "scheme_id": "uuid-here",
  "scheme_name": "HDFC Mid Cap Opportunities Fund",
  "fund_manager": "Chirag Setalvad",
  "inception_date": "2007-06-25",
  "benchmark": "NIFTY Midcap 100 TRI",
  "returns_1y": 18.4,
  "returns_3y": 22.1,
  "returns_5y": 19.8,
  "min_sip_amount": 500,
  "exit_load": "1% if redeemed within 1 year"
}
```

#### Step 6: Add Merge Node

1. Add a **Merge** node
2. Connect:
   - Input 1: output of `Fetch Latest NAV`
   - Input 2: output of `Fetch Scheme Details`
3. Mode: **Merge By Index** (both should have one item each)

#### Step 7: Add Set Node — Build Fund Profile

1. Add a **Set** node named `Build Fund Profile`
2. Mode: **Only Specified Fields**
3. Fields:
   - `fund_name`: `{{ $json.scheme_name }}`
   - `amc`: `{{ $json.amc_name }}`
   - `category`: `{{ $json.category }}`
   - `current_nav`: `{{ $json.nav }}`
   - `nav_date`: `{{ $json.date }}`
   - `expense_ratio_pct`: `{{ $json.expense_ratio }}`
   - `fund_manager`: `{{ $json.fund_manager }}`
   - `benchmark`: `{{ $json.benchmark }}`
   - `returns_1y_pct`: `{{ $json.returns_1y }}`
   - `returns_3y_pct`: `{{ $json.returns_3y }}`
   - `returns_5y_pct`: `{{ $json.returns_5y }}`
   - `exit_load_policy`: `{{ $json.exit_load }}`
   - `min_sip_amount`: `{{ $json.min_sip_amount }}`

#### Step 8: Add Respond to Webhook Node

1. Add a **Respond to Webhook** node
2. Response Body: `{{ JSON.stringify($json) }}`
3. Response Content-Type: `application/json`
4. Response Code: `200`

### Testing the Workflow

1. Click the Webhook node → **Listen for Test Event**
2. Open a terminal and run:
```bash
curl "http://localhost:5678/webhook-test/scheme-search?name=hdfc+mid+cap"
```
3. You should see the fund profile JSON in the response
4. Check each node's output panel in N8N to verify data at each step

---

## Part 5: Exercises

### Exercise 1: Search Three Different AMCs

Modify your webhook to accept an `amc` query parameter. Test with:
```bash
curl "http://localhost:5678/webhook-test/scheme-search?name=sbi+bluechip"
curl "http://localhost:5678/webhook-test/scheme-search?name=mirae+asset+emerging+bluechip"
curl "http://localhost:5678/webhook-test/scheme-search?name=axis+small+cap"
```

For each fund, note the expense ratio and 3-year returns in the output.

### Exercise 2: Compare Expense Ratios

Extend the workflow to fetch the top 5 results (not just the first), then use a Code node to:
1. Sort the results by `expense_ratio` ascending
2. Return the fund with the lowest expense ratio along with all 5 results ranked

Code node JavaScript:
```javascript
const items = $input.all();
const schemes = items.map(item => item.json);

// Sort by expense ratio ascending
const sorted = schemes.sort((a, b) => a.expense_ratio - b.expense_ratio);

return sorted.map(s => ({
  json: {
    rank_by_expense: sorted.indexOf(s) + 1,
    scheme_name: s.scheme_name,
    expense_ratio: s.expense_ratio,
    category: s.category,
    amc_name: s.amc_name
  }
}));
```

### Exercise 3: Fetch Historical NAV and Compute Returns

Fetch 30 days of NAV data instead of 1. Use a Code node to:
1. Find the oldest NAV in the 30-day window
2. Find today's NAV
3. Compute the 30-day return as a percentage

```javascript
const navData = $input.all().map(i => i.json);
const sorted = navData.sort((a, b) => new Date(a.date) - new Date(b.date));
const oldest = sorted[0].nav;
const latest = sorted[sorted.length - 1].nav;
const returnPct = ((latest - oldest) / oldest * 100).toFixed(2);

return [{
  json: {
    start_date: sorted[0].date,
    start_nav: oldest,
    end_date: sorted[sorted.length - 1].date,
    end_nav: latest,
    return_30d_pct: parseFloat(returnPct)
  }
}];
```

### Exercise 4: Build a Multi-Fund Comparison

Create a new workflow triggered by a webhook that accepts multiple fund names (comma-separated):
```bash
curl "http://localhost:5678/webhook-test/compare?funds=hdfc+mid+cap,sbi+bluechip,axis+small+cap"
```

Use a Code node to split the `funds` string, then use the **SplitInBatches** node to fetch each fund's profile in sequence, then **Merge** and return a comparison table.

---

## Part 6: Understanding N8N Error Handling

Before moving to Day 2, understand how N8N handles errors:

### Node-Level Errors
By default, if any node throws an error, the entire workflow execution fails and is marked as `Error`. You will see the failed node highlighted in red.

### Try/Catch with Error Workflow
For production workflows (Day 7 topic), you set an **Error Workflow** on any workflow. If execution fails, N8N automatically triggers the error workflow, passing error details. For now, know this exists.

### Retry on Fail
In any node's settings (gear icon), you can enable **Retry on Fail** with a configurable number of retries and wait between retries. Use this for HTTP nodes calling external APIs that may have transient failures.

### Testing vs Production Errors
In test mode (Listen for Test Event), errors show immediately in the UI. In production (workflow active), errors appear in the Execution Log (left sidebar → Executions).

---

## Day 1 Summary

You now know how to:
- Use Webhook, HTTP Request, Set, Code, Merge, and Respond to Webhook nodes
- Configure Header Auth credentials
- Chain API calls to enrich data progressively
- Test workflows interactively with Listen for Test Event
- Use expressions (`{{ $json.field }}`) to reference data between nodes

**Day 1 workflow to import:** `workflows/day1_scheme_explorer.json`

Tomorrow in Day 2 you will handle binary file uploads — sending a CAS PDF to the parser API and processing the structured portfolio data that comes back.
