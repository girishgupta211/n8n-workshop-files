# Day 6: Human-in-the-Loop Workflows

## Learning Objectives

By the end of Day 6 you will be able to:
- Articulate why human review is legally required in financial AI pipelines
- Implement N8N's Wait node with webhook resume for any approval gate
- Send approval requests via email with clickable Approve/Reject links
- Build Telegram inline button and Slack block-kit approval messages
- Build the Smart Redemption Advisor with a compliance gate
- Build the SIP Recommendation workflow with compliance review
- Handle timeout escalation when approval is not received within 24 hours
- Log all decisions (approved, rejected, timed-out) for audit purposes

---

## Part 1: Why Human Review Matters in Financial AI

### Fiduciary Responsibility

SEBI-registered investment advisors (RIAs) and mutual fund distributors (MFDs) have a fiduciary duty to their clients. This means:
- Recommendations must be suitable for the investor's risk profile and goals
- An AI system generating recommendations without human oversight may violate suitability requirements
- Any advice that leads to financial loss can create legal liability for the firm

### SEBI Regulatory Context

SEBI's regulations on AI in financial services (2024 guidelines) require:
- Human oversight for personalized investment advice
- Audit trails for every recommendation made to a client
- Ability to explain why a specific recommendation was made
- Client consent before acting on any AI-generated advice

An N8N workflow with a human approval gate provides:
1. A mandatory human review step before delivery
2. An immutable audit trail (every approval/rejection is logged with timestamp and reviewer)
3. A mechanism to override the AI's recommendation if it's inappropriate

### Types of Decisions Requiring Human Gates

| Decision Type | Risk Level | Who Approves |
|--------------|-----------|-------------|
| Redemption recommendation to investor | High | Compliance officer or senior advisor |
| SIP increase/decrease recommendation | Medium | Assigned advisor |
| Fund switch recommendation | Medium | Advisor |
| Portfolio rebalancing plan | High | Senior advisor + client consent |
| Tax harvesting plan | High | Tax advisor + compliance |
| New fund addition to portfolio | Low | Advisor |

---

## Part 2: N8N Wait Node Deep Dive

### How the Wait Node Works

The Wait node creates a "pause point" in a workflow execution:

1. When execution reaches the Wait node, N8N serializes the entire execution state to disk
2. The execution enters **Waiting** status — it consumes no CPU or memory while waiting
3. N8N generates a unique **Resume URL** for this specific execution instance
4. The workflow will remain paused until:
   - Someone calls the Resume URL (human action)
   - The configured timeout expires (automatic resume with timeout signal)

### Wait Node Configuration

1. Add a **Wait** node to your workflow
2. **Resume On**: 
   - `On Webhook Call`: resumes when someone hits the resume URL
   - `After Time Interval`: resumes after a fixed duration (for delays, not approvals)
   - `At Specified Time`: resumes at a specific datetime
3. **Webhook Method**: GET or POST (for approval links in emails, GET is easier — users click a link)
4. **Webhook Response Type**: 
   - `Last Node`: the entire sub-workflow after resume must return a response
   - `Response Node`: use a Respond to Webhook node at the end of the resume path

### The Resume URL

The resume URL is available during the email-building step via:
```
{{ $execution.resumeUrl }}
```

This URL looks like:
```
https://your-n8n.com/webhook/[workflow-id]/[execution-id]/waiting
```

You can append query parameters:
```
Approve: {{ $execution.resumeUrl }}?decision=approve&reviewer=compliance_team
Reject:  {{ $execution.resumeUrl }}?decision=reject&reason=unsuitable_risk_profile
```

When the reviewer clicks the link, the workflow resumes and `$json.query.decision` contains `approve` or `reject`.

### Execution State Persistence

- N8N stores paused executions in its database (SQLite or Postgres)
- All node output data up to the Wait node is preserved
- If N8N restarts, paused executions are correctly resumed when the resume URL is called
- Maximum wait time is 1 year by default (configurable)

---

## Part 3: Build — Smart Redemption Advisor with Human Gate

### Business Flow

1. Investor submits a redemption request via a form/webhook
2. AI Agent generates a smart redemption plan using `getSmartRedemptionPlan`
3. N8N pauses and emails the compliance officer with the AI's plan + Approve/Reject links
4. Compliance officer reviews and clicks a button
5. On approval: N8N stages the redemption via `stageBucketPurchaseCart` (or logs it)
6. On rejection: N8N logs the rejection and sends the investor a "we're reviewing your request" message
7. If no action in 24 hours: N8N escalates to the senior advisor

### Workflow Architecture

```
Webhook (POST /redemption-request)
    ↓
Set Node: extract request details
    ↓
AI Agent: getSmartRedemptionPlan
    ↓
Code Node: format plan summary
    ↓
Send Email: compliance officer (Approve/Reject links)
    ↓
Wait Node (24h timeout)
    ↓
IF: timed out?
    ├── YES → Send Escalation Email → Log Timeout → End
    └── NO  → IF: approved?
                   ├── YES → HTTP: stageBucketPurchaseCart → Email investor (confirmation) → Log Approval
                   └── NO  → Email investor (rejected/review) → Log Rejection
```

### Step-by-Step Build

#### Step 1: Webhook Trigger

1. Add a **Webhook** node
2. Method: **POST**
3. Path: `redemption-request`
4. Response Mode: `Using 'Respond to Webhook' Node`

Expected request body:
```json
{
  "investor_id": "inv_abc123",
  "investor_name": "Rahul Kumar",
  "investor_email": "rahul@example.com",
  "redemption_amount": 300000,
  "urgency": "high",
  "purpose": "medical_emergency",
  "notes": "Need within 3 business days"
}
```

#### Step 2: Set Node — Extract Request

```javascript
// Mode: Only Specified Fields
{
  investor_id: "{{ $json.body.investor_id }}",
  investor_name: "{{ $json.body.investor_name }}",
  investor_email: "{{ $json.body.investor_email }}",
  redemption_amount: "{{ $json.body.redemption_amount }}",
  urgency: "{{ $json.body.urgency || 'normal' }}",
  purpose: "{{ $json.body.purpose }}",
  notes: "{{ $json.body.notes || '' }}",
  request_id: "{{ 'REQ-' + Date.now() }}",
  submitted_at: "{{ $now }}"
}
```

#### Step 3: AI Agent — Generate Redemption Plan

1. Add an **AI Agent** node named `Generate Redemption Plan`
2. Prompt:
```
Generate a smart redemption plan for the following investor:

Investor: {{ $json.investor_name }} ({{ $json.investor_id }})
Redemption Amount Required: ₹{{ $json.redemption_amount.toLocaleString('en-IN') }}
Urgency: {{ $json.urgency }}
Purpose: {{ $json.purpose }}
Notes: {{ $json.notes }}

INSTRUCTIONS:
1. Call getPortfolioXray to understand the current portfolio
2. Call getSmartRedemptionPlan with redemption_amount={{ $json.redemption_amount }} and prefer_tax_efficiency=true
3. Provide:
   - Which funds to redeem from and how much
   - LTCG/STCG tax implications
   - Net amount after tax and exit loads
   - Expected settlement timeline (T+3 for equity, T+2 for liquid)
   - Alternative: suggest a partial liquidation if full redemption would trigger high tax
```

3. System Message: same portfolio advisor system prompt as Day 5
4. MCP Tools: portfolio-analytics and portfolio-cas servers

#### Step 4: Code Node — Format Plan Summary

```javascript
const request = $node['Extract Request'].json;
const agentOutput = $input.first().json.output;

// Extract key numbers from agent output using a simple regex scan
// In production you'd ask the AI to return structured JSON
const amountMatch = agentOutput.match(/net amount[:\s]+₹?([\d,]+)/i);
const netAmount = amountMatch ? amountMatch[1].replace(/,/g, '') : 'calculated in plan';

return [{
  json: {
    ...request,
    ai_plan: agentOutput,
    net_amount_estimated: netAmount,
    plan_generated_at: new Date().toISOString()
  }
}];
```

#### Step 5: Send Email to Compliance Officer

1. Add an **Email Send** node named `Email Compliance Team`
2. From: `n8n@yourfirm.com`
3. To: `compliance@yourfirm.com`
4. Subject: `[Approval Required] Redemption Request — {{ $json.investor_name }} — ₹{{ $json.redemption_amount.toLocaleString() }} — {{ $json.urgency.toUpperCase() }} priority`
5. HTML Body:

```html
<!DOCTYPE html>
<html>
<head>
<style>
  body { font-family: Arial, sans-serif; max-width: 700px; margin: 0 auto; padding: 20px; }
  .header { background: #1e3a5f; color: white; padding: 20px; border-radius: 8px 8px 0 0; }
  .content { border: 1px solid #e5e7eb; border-top: none; padding: 20px; border-radius: 0 0 8px 8px; }
  .detail-row { display: flex; margin-bottom: 8px; }
  .label { font-weight: bold; min-width: 200px; color: #374151; }
  .value { color: #1f2937; }
  .plan-box { background: #f3f4f6; border-left: 4px solid #3b82f6; padding: 16px; border-radius: 4px; margin: 16px 0; font-size: 14px; line-height: 1.6; white-space: pre-wrap; }
  .button { display: inline-block; padding: 14px 28px; border-radius: 6px; text-decoration: none; font-weight: bold; font-size: 16px; margin: 8px; }
  .approve { background: #22c55e; color: white; }
  .reject { background: #ef4444; color: white; }
  .footer { color: #9ca3af; font-size: 12px; margin-top: 20px; }
</style>
</head>
<body>
<div class="header">
  <h2 style="margin:0">Redemption Approval Required</h2>
  <p style="margin:4px 0 0">Request ID: {{ $json.request_id }}</p>
</div>
<div class="content">
  <div class="detail-row"><div class="label">Investor Name</div><div class="value">{{ $json.investor_name }}</div></div>
  <div class="detail-row"><div class="label">Investor ID</div><div class="value">{{ $json.investor_id }}</div></div>
  <div class="detail-row"><div class="label">Redemption Amount</div><div class="value">₹{{ $json.redemption_amount.toLocaleString('en-IN') }}</div></div>
  <div class="detail-row"><div class="label">Purpose</div><div class="value">{{ $json.purpose }}</div></div>
  <div class="detail-row"><div class="label">Urgency</div><div class="value">{{ $json.urgency }}</div></div>
  <div class="detail-row"><div class="label">Investor Notes</div><div class="value">{{ $json.notes || '(none)' }}</div></div>
  <div class="detail-row"><div class="label">Submitted At</div><div class="value">{{ $json.submitted_at }}</div></div>

  <h3>AI-Generated Redemption Plan</h3>
  <div class="plan-box">{{ $json.ai_plan }}</div>

  <h3>Your Decision</h3>
  <p>Please review the plan above and click one of the buttons below:</p>
  
  <a href="{{ $execution.resumeUrl }}?decision=approve&reviewer=compliance" class="button approve">APPROVE PLAN</a>
  <a href="{{ $execution.resumeUrl }}?decision=reject&reason=manual_review_required" class="button reject">REJECT PLAN</a>

  <p style="color:#6b7280; font-size:13px;">These links expire in 24 hours. If no action is taken, the request will be escalated to the senior advisor.</p>

  <div class="footer">
    This email was generated automatically by the Portfolio Advisory System. Request ID: {{ $json.request_id }}
  </div>
</div>
</body>
</html>
```

#### Step 6: Wait Node (24-Hour Timeout)

1. Add a **Wait** node
2. Resume On: **On Webhook Call**
3. Under **Limit Wait Time**: enabled
4. Wait Amount: `1440` minutes (24 hours)
5. When limit reached: continue execution (with timeout signal)

#### Step 7: IF Node — Timed Out?

1. Add an **IF** node named `Did It Time Out?`
2. Check: `{{ $json.query.decision }}` **is empty** (if timed out, no query params are set)

**True branch (timed out):**
- Send escalation email to senior advisor
- Log timeout to audit table
- Respond to original webhook: "Request escalated"

**False branch (decision received):**
- Proceed to next IF

#### Step 8: IF Node — Approved?

1. Add another **IF** node named `Was It Approved?`
2. Condition: `{{ $json.query.decision }}` equals `approve`

**True branch (approved):**
1. HTTP Request: `stageBucketPurchaseCart` via MCP (or log to DB)
2. Send Email to investor: confirmation with plan summary
3. Code Node: log approval record to audit trail

**False branch (rejected):**
1. Send Email to investor: "Your request is under review by our team"
2. Code Node: log rejection record with reason

### Audit Log Code Node (Both Paths)

```javascript
const request = $node['Extract Request'].json;
const decision = $json.query.decision;
const reviewer = $json.query.reviewer || 'compliance';
const reason = $json.query.reason || '';

return [{
  json: {
    audit_record: {
      request_id: request.request_id,
      investor_id: request.investor_id,
      investor_name: request.investor_name,
      redemption_amount: request.redemption_amount,
      decision: decision || 'timeout',
      reviewer,
      reason,
      decided_at: new Date().toISOString(),
      submitted_at: request.submitted_at,
      ai_plan_summary: request.ai_plan.slice(0, 500)
    }
  }
}];
```

---

## Part 4: Approval via Telegram Inline Button

For advisors who prefer Telegram, use inline keyboard buttons:

### Send Telegram with Inline Buttons

1. Add a **Telegram** node
2. Operation: **Send Message**
3. Chat ID: advisor's Telegram chat ID
4. Message:
```
🔔 Redemption Approval Required

Investor: {{ $json.investor_name }}
Amount: ₹{{ $json.redemption_amount.toLocaleString('en-IN') }}
Purpose: {{ $json.purpose }}
Request ID: {{ $json.request_id }}

The AI has generated a redemption plan. Please review and decide.
```
5. Under **Additional Fields** → **Reply Markup** → add JSON:
```json
{
  "inline_keyboard": [[
    {
      "text": "✅ APPROVE",
      "url": "{{ $execution.resumeUrl }}?decision=approve&reviewer=telegram"
    },
    {
      "text": "❌ REJECT",
      "url": "{{ $execution.resumeUrl }}?decision=reject&reviewer=telegram"
    }
  ]]
}
```

The advisor sees two inline buttons in Telegram. Clicking either one resumes the workflow.

---

## Part 5: Build — SIP Recommendation with Compliance Review

### Business Flow

1. Advisor creates a SIP recommendation for a client (via AI or manual input)
2. Compliance officer receives the recommendation for review
3. On approval: the recommendation is sent to the client via email/WhatsApp
4. On rejection: advisor is notified with feedback

### Simplified Workflow

```
Webhook (POST /sip-recommendation)
    ↓
AI Agent: getSipProjection + getBucketRecommendations
    ↓
Code Node: format recommendation
    ↓
Send Email: compliance (Approve/Reject)
    ↓
Wait Node (8h timeout)
    ↓
IF: Approved?
    ├── YES → Send Email to Client (recommendation)
    └── NO  → Send Email to Advisor (rejected with notes)
```

### Key Differences from Redemption Flow

- Timeout is 8 hours (business hours, not 24h)
- The email to the client is the output (not an API call)
- The reject path returns feedback to the originating advisor, not the client

### Client Email Template

```html
Subject: Personalized SIP Recommendation from [Your Firm Name]

Dear {{ $json.investor_name }},

Based on your financial goals and risk profile, our advisory team has reviewed 
the following SIP recommendation for you:

Fund: {{ $json.recommended_fund }}
Recommended Monthly SIP: ₹{{ $json.sip_amount.toLocaleString('en-IN') }}
Time Horizon: {{ $json.years }} years
Expected Corpus: ₹{{ $json.projected_value.toLocaleString('en-IN') }}

Why this recommendation:
{{ $json.rationale }}

IMPORTANT: This recommendation has been reviewed and approved by our compliance 
team on {{ $json.approval_date }}. Reference: {{ $json.request_id }}.

To start this SIP, please log in to your investor portal or contact your advisor.
[START SIP BUTTON]

Disclaimer: Mutual fund investments are subject to market risks. Past performance 
does not guarantee future returns. Please read all scheme-related documents carefully.

Regards,
[Your Firm Name] Advisory Team
```

---

## Part 6: Timeout Handling and Escalation

### Detecting a Timeout in the IF Node

When the Wait node times out, it resumes the workflow but does NOT add any query parameters. This means:
- After resume URL call: `$json.query.decision = "approve"` or `"reject"`
- After timeout: `$json.query` is empty or undefined

Check for timeout:
```javascript
// IF node condition
{{ Object.keys($json.query || {}).length === 0 }}
// OR
{{ !$json.query?.decision }}
```

### Escalation Email

```
Subject: [ESCALATION] Redemption Request Timeout — {{ $json.investor_name }} — {{ $json.request_id }}

This is an automated escalation alert.

A redemption approval request has been pending for more than 24 hours without a decision.

Request Details:
- Investor: {{ $json.investor_name }} ({{ $json.investor_id }})
- Amount: ₹{{ $json.redemption_amount.toLocaleString('en-IN') }}
- Submitted: {{ $json.submitted_at }}
- Original reviewer: compliance@yourfirm.com

ACTION REQUIRED:
Please review and approve or reject this request immediately.
[APPROVE LINK] [REJECT LINK]

This request has been waiting 24 hours. Client may be waiting for urgent funds.
```

---

## Part 7: Exercises

### Exercise 1: Slack Approval

Implement the approval gate using Slack's interactive messages:
1. Use the **Slack** node to send a message with Block Kit buttons
2. The buttons POST to a Slack interactivity URL you configure
3. Build a separate workflow that receives Slack's button callback and resumes the paused execution

This requires a Slack app with Interactivity enabled and a public N8N webhook URL.

### Exercise 2: Reason Required on Rejection

When the advisor rejects, require them to provide a reason. Instead of a simple link, redirect them to a small HTML form:

1. Approval link: `{{ $execution.resumeUrl }}?decision=approve`
2. Rejection link: link to a hosted form (or N8N form trigger) that captures the reason
3. The form then calls the resume URL with: `?decision=reject&reason=<encoded reason>`

### Exercise 3: Audit Dashboard

Build a separate workflow that:
1. Is triggered by a GET webhook at `/audit-dashboard`
2. Queries all approval/rejection records from the audit Postgres table for the past 30 days
3. Returns a JSON summary:
   - Total requests
   - Approval rate
   - Average time to decision
   - Most common rejection reasons
   - Timeout rate

---

## Day 6 Summary

You learned how to:
- Understand the regulatory and fiduciary requirement for human oversight in financial AI
- Implement the Wait node + webhook resume pattern for human approval gates
- Build email-based approvals with Approve/Reject HTML buttons
- Implement Telegram inline button approvals
- Handle 24-hour timeouts with automatic escalation
- Build a complete Smart Redemption Advisor with compliance gate
- Build a SIP Recommendation workflow with compliance review
- Create immutable audit trails for every decision

**Day 6 workflow to import:** `workflows/day6_human_approval.json`

Tomorrow in Day 7 you will build a pipeline evaluator to measure the quality of your AI advisor's responses, add production error handling, and assemble the capstone portfolio intelligence platform.
