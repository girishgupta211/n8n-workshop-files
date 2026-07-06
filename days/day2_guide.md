# Day 2: CAS PDF Parser — Parsing Real Portfolio Data

## Learning Objectives

By the end of Day 2 you will be able to:
- Explain what a CAS is and why it matters for portfolio analytics
- Upload a binary file (PDF) from N8N using the Read Binary File node
- Make a multipart/form-data POST request in N8N
- Process an array of holdings using SplitInBatches
- Compute portfolio summary statistics in a Code node
- Generate an HTML report and email it for human review before storing
- Implement a human-in-the-loop gate using the Wait node and webhook resume

---

## Part 1: What is a CAS?

### Consolidated Account Statement

A Consolidated Account Statement (CAS) is a single document that consolidates an investor's mutual fund holdings across all AMCs (Asset Management Companies) and registrar agencies (CAMS and KFintech). SEBI mandates that AMCs generate and email CAS to investors every month.

The CAS includes:
- Investor details (PAN, name, email)
- Holdings per scheme (units held, cost of acquisition, current NAV, current value)
- Transaction history (purchases, redemptions, switches, dividends)
- Folios from both CAMS and KFintech registrars

### Why CAS Parsing Matters

Raw CAS PDFs are not machine-readable — they are generated PDFs with text extracted from multiple data sources. Parsing them requires:
1. PDF text extraction with layout preservation
2. Identifying folio boundaries and scheme blocks
3. Parsing transaction tables and computing XIRR

The MF Engine CAS parser does all of this for you via a single API call.

### Getting a Sample CAS

For the workshop, you can:
1. Download your own CAS from CAMS (camsonline.com) or KFintech (kfintech.com)
2. Use a sample redacted CAS PDF provided by the workshop organizer
3. Generate a test CAS from the staging environment

---

## Part 2: The CAS Parser API

### Endpoint

```
POST https://app2.mfapis.club/api/v2/cas
Content-Type: multipart/form-data
x-api-key: <your-api-key>
```

### Request Format

The request body must be `multipart/form-data` with:
- Field `file`: the CAS PDF binary
- Field `password` (optional): PDF password if the CAS is password-protected (CAMS uses PAN-based passwords)

### Response Format

```json
{
  "pan": "ABCDE1234F",
  "name": "Rahul Kumar",
  "email": "rahul@example.com",
  "statement_period": {
    "from": "2025-01-01",
    "to": "2025-06-30"
  },
  "holdings": [
    {
      "scheme_id": "uuid-here",
      "scheme_name": "HDFC Mid Cap Opportunities Fund - Growth",
      "amc_name": "HDFC Mutual Fund",
      "folio_number": "1234567/89",
      "units": 1523.456,
      "nav": 98.432,
      "current_value": 150008.34,
      "purchase_cost": 120000.00,
      "xirr": 18.4,
      "category": "Mid Cap",
      "sub_category": "Mid Cap Fund"
    },
    {
      "scheme_name": "SBI Bluechip Fund - Growth",
      "amc_name": "SBI Mutual Fund",
      "folio_number": "9876543/21",
      "units": 892.123,
      "nav": 72.18,
      "current_value": 64386.59,
      "purchase_cost": 55000.00,
      "xirr": 12.1,
      "category": "Large Cap",
      "sub_category": "Large Cap Fund"
    }
  ],
  "summary": {
    "total_invested": 175000.00,
    "current_value": 214394.93,
    "overall_xirr": 16.2,
    "absolute_returns_pct": 22.5,
    "total_schemes": 2,
    "total_folios": 2
  }
}
```

### Password Logic

For CAMS CAS: the default password is the investor's PAN (uppercase). For KFintech CAS: it is the first 8 characters of the PAN + date of birth in DDMMYYYY format. If no password, leave the `password` field empty.

---

## Part 3: Binary File Upload in N8N

### The Read Binary File Node

N8N has a `readBinaryFile` node that reads a file from the filesystem where N8N is running. For self-hosted N8N, this means the local server filesystem.

**Configuration:**
- File Path: absolute path to the PDF file
  - Example: `/home/n8n/uploads/sample_cas.pdf`
  - For Docker installs: `/data/uploads/sample_cas.pdf` (mount a volume)

**Output:** the node outputs a binary item. The binary data is stored under a key (default: `data`). The item JSON will have a `binary` property with metadata, but the actual bytes are in the binary attachment.

### Accessing the Binary in HTTP Request

When the HTTP Request node receives an item with binary data attached, you configure the body as **Form Data** and reference the binary:

1. Set **Send Body**: enabled
2. Body Content Type: **Form Data**
3. Add a form field:
   - Type: **File**
   - Field Name: `file`
   - Data: select **From Binary** and specify the binary key (e.g., `data`)

N8N automatically sets the correct `Content-Type: multipart/form-data; boundary=...` header.

### Handling Password-Protected PDFs

Add a second form field:
- Type: **Text**
- Field Name: `password`
- Value: `{{ $json.query.password || '' }}`

If the caller passes `?password=ABCDE1234F` to your webhook, this gets forwarded to the parser.

---

## Part 4: Build — CAS Upload Pipeline

### Workflow Architecture

```
Manual Trigger
    ↓
Read Binary File (CAS PDF)
    ↓
HTTP Request: POST /api/v2/cas (multipart)
    ↓
Code Node: compute summary statistics
    ↓
Code Node: generate HTML report
    ↓
Send Email: advisor review (human-in-the-loop gate)
    ↓
Wait Node (wait for advisor approval via webhook)
    ↓
IF Node: approved?
    ├── YES → Set Node (format final record) → Respond to Webhook
    └── NO  → Set Node (log rejection) → Respond to Webhook
```

### Step-by-Step Build

#### Step 1: Manual Trigger

For Day 2, use a **Manual Trigger** node to kick off the workflow manually. In production you would replace this with a webhook that accepts the CAS file upload.

#### Step 2: Read Binary File

1. Add a **Read Binary File** node
2. File Path: `/home/n8n/uploads/sample_cas.pdf`
   - Adjust to wherever you put the sample CAS on your N8N server
3. Property Name: `data` (the binary data key)

#### Step 3: HTTP Request — POST to CAS Parser

1. Add an **HTTP Request** node named `Parse CAS PDF`
2. Method: **POST**
3. URL: `https://app2.mfapis.club/api/v2/cas`
4. Authentication: **MF Engine API**
5. Under **Send Body**: enabled
6. Body Content Type: **Form Data**
7. Body Parameters:
   - Parameter 1:
     - Type: **File**
     - Name: `file`
     - Input Data Field Name: `data`
   - Parameter 2 (optional):
     - Type: **Text**
     - Name: `password`
     - Value: `` (empty, or expression if password known)
8. Under **Options** → **Response**: set Response Format to `JSON`

#### Step 4: Code Node — Compute Portfolio Summary

1. Add a **Code** node named `Compute Portfolio Summary`
2. Mode: **Run Once for All Items**
3. JavaScript:

```javascript
const casData = $input.first().json;

const holdings = casData.holdings || [];

// Compute per-holding gain/loss
const enrichedHoldings = holdings.map(h => {
  const absoluteGain = h.current_value - h.purchase_cost;
  const gainPct = ((absoluteGain / h.purchase_cost) * 100).toFixed(2);
  return {
    ...h,
    absolute_gain: Math.round(absoluteGain * 100) / 100,
    gain_pct: parseFloat(gainPct),
    is_profit: absoluteGain >= 0
  };
});

// Sort by current value descending
const sortedHoldings = enrichedHoldings.sort((a, b) => b.current_value - a.current_value);

// Top and bottom performers by XIRR
const byXirr = [...enrichedHoldings].sort((a, b) => b.xirr - a.xirr);
const topPerformer = byXirr[0];
const bottomPerformer = byXirr[byXirr.length - 1];

// Category breakdown
const categoryMap = {};
enrichedHoldings.forEach(h => {
  if (!categoryMap[h.category]) categoryMap[h.category] = { value: 0, invested: 0, count: 0 };
  categoryMap[h.category].value += h.current_value;
  categoryMap[h.category].invested += h.purchase_cost;
  categoryMap[h.category].count++;
});

const totalValue = casData.summary.current_value;
const categoryBreakdown = Object.entries(categoryMap).map(([cat, data]) => ({
  category: cat,
  current_value: Math.round(data.value * 100) / 100,
  invested: Math.round(data.invested * 100) / 100,
  allocation_pct: Math.round((data.value / totalValue) * 10000) / 100,
  scheme_count: data.count
}));

return [{
  json: {
    investor_name: casData.name,
    pan: casData.pan,
    statement_period: casData.statement_period,
    summary: casData.summary,
    holdings: sortedHoldings,
    category_breakdown: categoryBreakdown,
    top_performer: topPerformer,
    bottom_performer: bottomPerformer,
    generated_at: new Date().toISOString()
  }
}];
```

#### Step 5: Code Node — Generate HTML Report

1. Add a **Code** node named `Generate HTML Report`
2. Mode: **Run Once per Item**
3. JavaScript:

```javascript
const data = $input.item.json;
const { investor_name, pan, summary, holdings, category_breakdown, top_performer, bottom_performer } = data;

const formatCurrency = (n) => '₹' + n.toLocaleString('en-IN', { maximumFractionDigits: 2 });
const formatPct = (n) => n.toFixed(2) + '%';

const holdingsRows = holdings.map(h => `
  <tr>
    <td>${h.scheme_name}</td>
    <td>${h.amc_name}</td>
    <td>${h.category}</td>
    <td>${h.units.toFixed(3)}</td>
    <td>${formatCurrency(h.nav)}</td>
    <td>${formatCurrency(h.current_value)}</td>
    <td>${formatCurrency(h.purchase_cost)}</td>
    <td style="color:${h.is_profit ? '#22c55e' : '#ef4444'}">${formatCurrency(h.absolute_gain)}</td>
    <td>${formatPct(h.gain_pct)}</td>
    <td>${formatPct(h.xirr)}</td>
  </tr>
`).join('');

const categoryRows = category_breakdown.map(c => `
  <tr>
    <td>${c.category}</td>
    <td>${c.scheme_count}</td>
    <td>${formatCurrency(c.current_value)}</td>
    <td>${formatPct(c.allocation_pct)}</td>
  </tr>
`).join('');

const html = `
<!DOCTYPE html>
<html>
<head>
  <meta charset="utf-8">
  <title>Portfolio Report — ${investor_name}</title>
  <style>
    body { font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif; margin: 0; padding: 24px; color: #1a1a1a; background: #f9fafb; }
    .card { background: white; border-radius: 12px; padding: 24px; margin-bottom: 24px; box-shadow: 0 1px 3px rgba(0,0,0,0.1); }
    h1 { font-size: 24px; margin: 0 0 4px; }
    .subtitle { color: #6b7280; font-size: 14px; margin-bottom: 24px; }
    .stats-grid { display: grid; grid-template-columns: repeat(4, 1fr); gap: 16px; }
    .stat { background: #f3f4f6; border-radius: 8px; padding: 16px; }
    .stat-label { font-size: 12px; color: #6b7280; text-transform: uppercase; letter-spacing: 0.05em; }
    .stat-value { font-size: 22px; font-weight: 700; margin-top: 4px; }
    .positive { color: #22c55e; }
    .negative { color: #ef4444; }
    table { width: 100%; border-collapse: collapse; font-size: 13px; }
    th { background: #f3f4f6; padding: 10px 12px; text-align: left; font-weight: 600; border-bottom: 2px solid #e5e7eb; }
    td { padding: 10px 12px; border-bottom: 1px solid #f3f4f6; }
    tr:hover td { background: #fafafa; }
    h2 { font-size: 18px; margin: 0 0 16px; }
    .highlight { background: #fef9c3; border-left: 4px solid #eab308; padding: 12px 16px; border-radius: 4px; margin: 8px 0; font-size: 14px; }
  </style>
</head>
<body>
  <div class="card">
    <h1>Portfolio Report</h1>
    <div class="subtitle">Investor: ${investor_name} &nbsp;|&nbsp; PAN: ${pan} &nbsp;|&nbsp; Generated: ${new Date().toLocaleDateString('en-IN')}</div>
    
    <div class="stats-grid">
      <div class="stat">
        <div class="stat-label">Total Invested</div>
        <div class="stat-value">${formatCurrency(summary.total_invested)}</div>
      </div>
      <div class="stat">
        <div class="stat-label">Current Value</div>
        <div class="stat-value positive">${formatCurrency(summary.current_value)}</div>
      </div>
      <div class="stat">
        <div class="stat-label">Absolute Gain</div>
        <div class="stat-value ${summary.current_value >= summary.total_invested ? 'positive' : 'negative'}">
          ${formatCurrency(summary.current_value - summary.total_invested)}
        </div>
      </div>
      <div class="stat">
        <div class="stat-label">Overall XIRR</div>
        <div class="stat-value ${summary.overall_xirr >= 0 ? 'positive' : 'negative'}">${formatPct(summary.overall_xirr)}</div>
      </div>
    </div>
  </div>

  <div class="card">
    <div class="highlight">Top Performer: <strong>${top_performer?.scheme_name}</strong> — XIRR: ${formatPct(top_performer?.xirr || 0)}</div>
    <div class="highlight">Bottom Performer: <strong>${bottom_performer?.scheme_name}</strong> — XIRR: ${formatPct(bottom_performer?.xirr || 0)}</div>
  </div>

  <div class="card">
    <h2>Holdings (${summary.total_schemes} schemes)</h2>
    <table>
      <thead>
        <tr>
          <th>Scheme</th><th>AMC</th><th>Category</th><th>Units</th><th>NAV</th>
          <th>Current Value</th><th>Invested</th><th>Gain/Loss</th><th>Return %</th><th>XIRR</th>
        </tr>
      </thead>
      <tbody>${holdingsRows}</tbody>
    </table>
  </div>

  <div class="card">
    <h2>Category Breakdown</h2>
    <table>
      <thead>
        <tr><th>Category</th><th>Schemes</th><th>Value</th><th>Allocation</th></tr>
      </thead>
      <tbody>${categoryRows}</tbody>
    </table>
  </div>
</body>
</html>
`;

return {
  json: {
    ...data,
    html_report: html
  }
};
```

#### Step 6: Send Email — Advisor Review

1. Add an **Email Send** node named `Email Advisor for Review`
2. Configure SMTP credentials (Settings → Credentials → SMTP → add your details)
3. To: `advisor@yourfirm.com`
4. Subject: `[Review Required] Portfolio CAS parsed for {{ $json.investor_name }} ({{ $json.pan }})`
5. HTML Body: `{{ $json.html_report }}`
6. Include a plain text note in the email body:
   ```
   Please review the attached portfolio analysis for {{ $json.investor_name }}.
   
   Click APPROVE to store in database: {{ $execution.resumeUrl }}?action=approve
   Click REJECT to discard: {{ $execution.resumeUrl }}?action=reject
   
   This link expires in 24 hours.
   ```

> **Key concept:** `{{ $execution.resumeUrl }}` is the magic URL that will resume the paused workflow when called.

#### Step 7: Wait Node

1. Add a **Wait** node after the Email node
2. Resume Type: **On Webhook Call**
3. This generates a unique resume URL for each execution
4. Set a timeout: **1440 minutes** (24 hours) — after this the workflow auto-resumes with a timeout signal

#### Step 8: IF Node — Check Decision

1. Add an **IF** node named `Check Advisor Decision`
2. Condition: `{{ $json.query.action }}` equals `approve`
3. True branch → store/return approved data
4. False branch → log rejection

#### Step 9: Final Set Nodes

**Approved path:**
```javascript
// Set node: "Format Approved Record"
{
  status: "approved",
  investor_name: "{{ $json.investor_name }}",
  pan: "{{ $json.pan }}",
  summary: "{{ $json.summary }}",
  approved_at: "{{ $now }}"
}
```

**Rejected path:**
```javascript
// Set node: "Log Rejection"
{
  status: "rejected",
  investor_name: "{{ $json.investor_name }}",
  pan: "{{ $json.pan }}",
  rejected_at: "{{ $now }}"
}
```

---

## Part 5: Processing Multiple Holdings with SplitInBatches

When you need to call another API for each holding (e.g., fetch the scheme details for each holding), use **SplitInBatches**.

### SplitInBatches Pattern

```
Code Node (output: array of holdings as items)
    ↓
SplitInBatches (batch size: 1)
    ↓
HTTP Request (scheme details for each holding)
    ↓
Set Node (enrich holding with fetched data)
    ↓
[back to SplitInBatches loop]
    ↓ (when all processed)
Merge (aggregate all enriched holdings)
```

### Code to Split Holdings Into Items

Before SplitInBatches, use a Code node to convert the holdings array into individual items:

```javascript
const casData = $input.first().json;
return casData.holdings.map(h => ({ json: h }));
```

Now SplitInBatches processes one holding at a time, making an HTTP call for each.

### Merge Back Into One Item

After SplitInBatches finishes looping, add a Code node to collect all enriched holdings:

```javascript
const allHoldings = $input.all().map(item => item.json);
return [{
  json: {
    enriched_holdings: allHoldings,
    total_schemes: allHoldings.length
  }
}];
```

---

## Part 6: Understanding the Wait Node / Human Gate Pattern

The Wait node is N8N's core mechanism for human-in-the-loop workflows. Here is how it works:

### Execution Flow

1. Workflow runs until it hits the Wait node
2. The current execution state is **serialized to disk** — N8N saves all the data from every previous node
3. The workflow execution status becomes **Waiting**
4. N8N generates a unique resume URL for this specific execution
5. You send that URL to a human (via email, Slack, SMS)
6. When the human clicks the URL, N8N receives a webhook call
7. The execution is **deserialized from disk** and continues from the Wait node
8. The webhook call's query parameters (like `?action=approve`) are available in `$json` downstream

### Resume URL

The resume URL looks like:
```
https://your-n8n.com/webhook/[workflow-id]/[execution-id]
```

In the email body, you embed this URL with your parameters:
```
Approve: https://your-n8n.com/webhook/abc123/exec456?action=approve
Reject:  https://your-n8n.com/webhook/abc123/exec456?action=reject
```

### Important Notes
- The resume URL is **single-use** — once called, it resumes the workflow and is invalidated
- N8N stores paused executions in its database (SQLite or Postgres)
- If you restart N8N, paused executions are resumed when it comes back online
- The 24-hour timeout is configurable per Wait node

---

## Part 7: Exercises

### Exercise 1: Add PDF Password Support

Modify the workflow to accept a password parameter:
1. Change the trigger to a Webhook (GET)
2. Accept `?password=ABCDE1234F` in the query string
3. Forward it to the CAS parser API

### Exercise 2: Add Category-Level Alerts

In the Compute Summary Code node, add logic to flag if:
- Any single category represents more than 60% of the portfolio (concentration risk)
- The overall XIRR is below 8% (underperformance vs FD rates)

Include these flags in the email subject:
```
[ALERT] [Review Required] Portfolio CAS — concentration risk detected for Rahul Kumar
```

### Exercise 3: Store to a Spreadsheet

Instead of (or in addition to) the email review, use the **Google Sheets** node to append the portfolio summary row to a tracking spreadsheet. Columns: date, investor_name, pan, total_invested, current_value, xirr.

---

## Day 2 Summary

You learned how to:
- Read binary files from the filesystem in N8N
- POST multipart/form-data to the CAS parser API
- Process complex nested JSON responses with Code nodes
- Generate HTML reports dynamically
- Implement human-in-the-loop approval using the Wait node and resume URLs
- Iterate over arrays with SplitInBatches

**Day 2 workflow to import:** `workflows/day2_cas_parser.json`

Tomorrow in Day 3 you will schedule automated portfolio health checks using portfolio Xray and stress testing APIs, and store time-series data to a database.
