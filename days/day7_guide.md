# Day 7: Pipeline Evaluation & Production Hardening

## Learning Objectives

By the end of Day 7 you will be able to:
- Explain why evaluating AI pipelines is as important as building them
- Build a pipeline evaluator using reference answers and rubric scoring
- Implement reference answer comparison and semantic quality scoring
- Store evaluation results in Airtable or Postgres
- Add Try/Catch error handling with exponential backoff retry
- Implement dead-letter webhook patterns for failed executions
- Set up structured logging and SLA tracking
- Assemble the full capstone AI Portfolio Intelligence Platform

---

## Part 1: Why Evaluate AI Pipelines?

### The Problem with "It Looks Good" Testing

When you test your AI Portfolio Advisor (Day 5/6) manually, you ask a few questions, the answers look reasonable, and you declare it working. But:

- **Hallucination risk**: the AI might confidently state the wrong XIRR or recommend a fund that doesn't exist
- **Stale data**: if the MCP tool fails silently, the AI responds with outdated or cached data
- **Wrong tool selection**: the AI might call `getPortfolioXray` for a SIP question instead of `getSipProjection`, giving a technically correct but irrelevant answer
- **Regression after changes**: updating the system prompt might fix one query type but break another

### What to Measure

| Metric | Description | How to Measure |
|--------|-------------|----------------|
| **Accuracy** | Are the numbers correct? | Compare against known values from the API |
| **Tool Selection** | Did the AI call the right tools? | Check `intermediateSteps` for tool names |
| **Completeness** | Did the answer address all parts of the question? | Rubric checklist |
| **Hallucination Score** | Did the AI invent numbers not in the API response? | Cross-check key numbers against raw API output |
| **Latency** | How long did the response take? | Timestamp diff between start and completion |
| **Cost** | How many tokens were consumed? | Track from OpenAI usage API |
| **Tool Efficiency** | Did the AI use more tools than necessary? | Count tool calls |

### Test Dataset Design

A good evaluation dataset has:
1. **Coverage**: tests each tool at least once
2. **Edge cases**: questions where the AI might fail (ambiguous requests, multi-part questions, corner cases)
3. **Reference answers**: ground-truth answers from the actual API (not AI-generated)
4. **Rubrics**: clear criteria for what makes an answer good or bad

---

## Part 2: Build — Pipeline Evaluator

### Evaluation Dataset

Define 5–10 test cases with questions, expected tool calls, and reference answers.

**Test Case Format:**
```json
{
  "test_id": "TC001",
  "category": "health_check",
  "question": "What is my current portfolio health score and what are the main risk flags?",
  "expected_tools": ["getPortfolioHealthScore", "getPortfolioXray"],
  "reference_answer": {
    "must_contain": ["health score", "diversification", "concentration"],
    "must_mention_score": true,
    "must_mention_flags": true,
    "acceptable_score_range": [0, 100]
  },
  "rubric": {
    "mentions_numeric_health_score": 1,
    "identifies_at_least_one_flag": 1,
    "recommends_action": 1,
    "uses_correct_tool": 1,
    "no_hallucinated_numbers": 1
  },
  "max_score": 5
}
```

### Workflow Architecture

```
Manual Trigger
    ↓
Code Node: define 5 test cases
    ↓
SplitInBatches (batch size: 1)
    ↓
AI Agent: run test question
    ↓
Code Node: evaluate response against rubric
    ↓
[back to SplitInBatches loop]
    ↓ (when all done)
Code Node: aggregate scores + build report
    ↓
HTTP Request: store to Airtable (or Postgres)
    ↓
Send Email: quality report to team
```

### Step-by-Step Build

#### Step 1: Manual Trigger

Add a **Manual Trigger** node. This evaluator is run on-demand (though you could schedule it weekly).

#### Step 2: Code Node — Define Test Cases

```javascript
const testCases = [
  {
    test_id: "TC001",
    category: "health_check",
    question: "What is my current portfolio health score? List all risk flags.",
    expected_tools: ["getPortfolioHealthScore"],
    rubric: {
      mentions_score: "\\d+/100|score.*\\d+|\\d+.*score",
      mentions_flags: "flag|risk|concentration|concern",
      gives_recommendation: "recommend|suggest|consider|should",
      uses_rupee_symbol: "₹|lakh|crore",
      no_generic_response: "based on your portfolio|your portfolio"
    }
  },
  {
    test_id: "TC002",
    category: "stress_test",
    question: "How would my portfolio have performed in the 2008 global financial crisis? What was the estimated drawdown?",
    expected_tools: ["getPortfolioStressTest"],
    rubric: {
      mentions_gfc: "2008|financial crisis|GFC",
      mentions_drawdown: "drawdown|fall|drop|decline|lost",
      mentions_percentage: "\\d+%|percent",
      mentions_recovery: "recover|months|years",
      uses_specific_numbers: "\\d+\\.\\d+"
    }
  },
  {
    test_id: "TC003",
    category: "tax_redemption",
    question: "I need Rs 2 lakhs for a medical emergency. Which funds should I redeem to minimize tax? Show me the tax breakdown.",
    expected_tools: ["getSmartRedemptionPlan", "getTaxAwareRedemptionPlan"],
    rubric: {
      mentions_ltcg: "LTCG|long.term capital gain|long term",
      mentions_stcg: "STCG|short.term capital gain|short term",
      mentions_specific_fund: "fund|scheme",
      mentions_net_amount: "net amount|after tax|net",
      mentions_units: "units|redemption"
    }
  },
  {
    test_id: "TC004",
    category: "sip_projection",
    question: "If I invest Rs 20,000 per month in HDFC Mid Cap for 10 years assuming 13% returns, what corpus will I accumulate?",
    expected_tools: ["getSipProjection"],
    rubric: {
      mentions_corpus: "corpus|total|accumulated|value",
      mentions_invested: "invested|invest|₹|lakh",
      mentions_gain: "gain|profit|return|growth",
      provides_calculation: "\\d+,\\d+|lakh|crore",
      mentions_time: "10 year|decade"
    }
  },
  {
    test_id: "TC005",
    category: "fund_comparison",
    question: "Compare HDFC Mid Cap Opportunities Fund and Nippon India Growth Fund. Which one has better alpha generation?",
    expected_tools: ["comparePortfolioFunds", "getFundVsIndexComparison"],
    rubric: {
      compares_both_funds: "HDFC.*Nippon|Nippon.*HDFC",
      mentions_alpha: "alpha|benchmark|outperform",
      gives_recommendation: "recommend|prefer|better|switch|stay",
      mentions_returns: "return|XIRR|%",
      structured_comparison: "vs|compared|versus|better than|higher"
    }
  }
];

return testCases.map(tc => ({ json: tc }));
```

#### Step 3: SplitInBatches Node

1. Add a **SplitInBatches** node
2. Batch Size: `1`
3. Connect Code Node → SplitInBatches

#### Step 4: AI Agent — Run Test Question

1. Add an **AI Agent** node named `Test AI Advisor`
2. Prompt: `{{ $json.question }}`
3. Same system prompt and MCP tools as Day 5
4. Save `test_id` and `rubric` from the current item — you'll need them after the agent runs
5. Set Max Iterations: `8`

#### Step 5: Code Node — Evaluate Response

```javascript
const testCase = $node['SplitInBatches'].json; // the test case data
const agentResponse = $input.first().json;
const aiAnswer = agentResponse.output || '';
const toolsUsed = (agentResponse.intermediateSteps || [])
  .map(s => s.action?.tool)
  .filter(Boolean);

const rubric = testCase.rubric;
const expectedTools = testCase.expected_tools;

// Score rubric criteria
const rubricResults = {};
let rubricScore = 0;
const maxRubricScore = Object.keys(rubric).length;

Object.entries(rubric).forEach(([criterion, pattern]) => {
  const regex = new RegExp(pattern, 'i');
  const passed = regex.test(aiAnswer);
  rubricResults[criterion] = passed;
  if (passed) rubricScore++;
});

// Score tool selection
const toolsCorrect = expectedTools.every(t => toolsUsed.includes(t));
const unexpectedTools = toolsUsed.filter(t => !expectedTools.includes(t));
const toolScore = toolsCorrect ? 1 : 0;

// Compute total score (0-5 scale)
const totalScore = Math.round((rubricScore / maxRubricScore) * 4 + toolScore);

// Determine pass/fail
const passed = totalScore >= 3;

// Count tokens (approximate: 1 token ≈ 4 chars)
const approxTokensUsed = Math.round((aiAnswer.length + testCase.question.length) / 4);

return [{
  json: {
    test_id: testCase.test_id,
    category: testCase.category,
    question: testCase.question,
    ai_answer: aiAnswer.slice(0, 1000), // truncate for storage
    tools_used: toolsUsed,
    expected_tools: expectedTools,
    tools_correct: toolsCorrect,
    unexpected_tools: unexpectedTools,
    rubric_results: rubricResults,
    rubric_score: rubricScore,
    max_rubric_score: maxRubricScore,
    tool_score: toolScore,
    total_score: totalScore,
    max_score: 5,
    passed,
    score_pct: Math.round((totalScore / 5) * 100),
    approx_tokens: approxTokensUsed,
    evaluated_at: new Date().toISOString()
  }
}];
```

#### Step 6: Aggregate All Results

After SplitInBatches completes all iterations, add a **Code** node (connected to SplitInBatches Done output):

```javascript
const results = $input.all().map(i => i.json);

const totalTests = results.length;
const passed = results.filter(r => r.passed).length;
const failed = results.filter(r => !r.passed).length;
const avgScore = results.reduce((s, r) => s + r.total_score, 0) / totalTests;
const avgRubricScore = results.reduce((s, r) => s + r.score_pct, 0) / totalTests;
const totalTokens = results.reduce((s, r) => s + r.approx_tokens, 0);

const toolAccuracy = results.filter(r => r.tools_correct).length / totalTests;

// Per-category breakdown
const byCategory = {};
results.forEach(r => {
  if (!byCategory[r.category]) byCategory[r.category] = { tests: 0, passed: 0, total_score: 0 };
  byCategory[r.category].tests++;
  if (r.passed) byCategory[r.category].passed++;
  byCategory[r.category].total_score += r.total_score;
});

const categoryBreakdown = Object.entries(byCategory).map(([cat, data]) => ({
  category: cat,
  tests: data.tests,
  pass_rate_pct: Math.round((data.passed / data.tests) * 100),
  avg_score: Math.round((data.total_score / data.tests) * 10) / 10
}));

return [{
  json: {
    run_id: 'EVAL-' + Date.now(),
    run_date: new Date().toISOString().slice(0, 10),
    summary: {
      total_tests: totalTests,
      passed,
      failed,
      pass_rate_pct: Math.round((passed / totalTests) * 100),
      avg_score_out_of_5: Math.round(avgScore * 10) / 10,
      avg_rubric_score_pct: Math.round(avgRubricScore),
      tool_selection_accuracy_pct: Math.round(toolAccuracy * 100),
      total_tokens_consumed: totalTokens
    },
    category_breakdown: categoryBreakdown,
    individual_results: results,
    overall_grade: avgScore >= 4 ? 'A' : avgScore >= 3 ? 'B' : avgScore >= 2 ? 'C' : 'D'
  }
}];
```

#### Step 7: Store to Airtable

1. Add an **HTTP Request** node named `Store to Airtable`
2. Method: **POST**
3. URL: `https://api.airtable.com/v0/YOUR_BASE_ID/EvaluationRuns`
4. Headers:
   - `Authorization`: `Bearer YOUR_AIRTABLE_API_KEY`
   - `Content-Type`: `application/json`
5. Body (JSON):
```json
{
  "fields": {
    "Run ID": "{{ $json.run_id }}",
    "Run Date": "{{ $json.run_date }}",
    "Pass Rate %": "{{ $json.summary.pass_rate_pct }}",
    "Average Score": "{{ $json.summary.avg_score_out_of_5 }}",
    "Tool Accuracy %": "{{ $json.summary.tool_selection_accuracy_pct }}",
    "Overall Grade": "{{ $json.overall_grade }}",
    "Total Tokens": "{{ $json.summary.total_tokens_consumed }}",
    "Results JSON": "{{ JSON.stringify($json.individual_results).slice(0, 100000) }}"
  }
}
```

#### Step 8: Email Quality Report

Send a formatted HTML email with the evaluation summary to the engineering team.

---

## Part 3: Production Error Handling

### Try/Catch with Error Workflows

N8N's **Try/Catch** node (available in v1.40+) wraps a subgraph. If any node inside the Try block throws an error, execution passes to the Catch block instead of crashing the workflow.

```
Trigger
    ↓
[Try Block Start]
    HTTP Request (may fail)
    Code Node (may throw)
    AI Agent (may timeout)
[Try Block End — Catch Block Start]
    Log Error
    Send Alert
    Store to Dead Letter Queue
[Catch Block End]
    ↓
Continue (or Stop)
```

If you do not have the Try/Catch node, use **Continue on Error** mode on individual HTTP Request nodes:
1. Click the gear icon on any HTTP Request node
2. Enable **Continue on Fail**
3. The node passes the error as JSON to the next node
4. Use an IF node to check for errors: `{{ $json.error !== undefined }}`

### Exponential Backoff Retry

For transient API failures (rate limits, 503s), implement retry logic:

```javascript
// Code node: retry with exponential backoff
async function retryWithBackoff(fn, maxRetries = 3, baseDelayMs = 1000) {
  let lastError;
  for (let attempt = 0; attempt < maxRetries; attempt++) {
    try {
      return await fn();
    } catch (err) {
      lastError = err;
      if (attempt < maxRetries - 1) {
        const delay = baseDelayMs * Math.pow(2, attempt); // 1s, 2s, 4s
        await new Promise(resolve => setTimeout(resolve, delay));
      }
    }
  }
  throw lastError;
}
```

In N8N, use the **Retry on Fail** setting on HTTP Request nodes:
1. Click the gear icon
2. Enable **Retry on Fail**
3. Max tries: `3`
4. Wait Between Tries: `Exponential` or set manually (1000ms, 2000ms, 4000ms)

### Dead-Letter Webhook

When a workflow fails fatally (error workflow triggered), log the failure to a "dead letter" endpoint:

1. Create an **Error Workflow** in N8N:
   - Settings → add "Error Workflow" → select your error handling workflow
2. The error workflow receives:
   ```json
   {
     "execution": { "id": "...", "url": "...", "error": { "message": "...", "stack": "..." } },
     "workflow": { "id": "...", "name": "..." }
   }
   ```
3. In the error workflow:
   - Send Telegram alert: "Workflow {{ $json.workflow.name }} failed: {{ $json.execution.error.message }}"
   - Log to Postgres: INSERT into workflow_errors table
   - Optionally: POST to a Slack channel

**Dead-Letter Log Table:**
```sql
CREATE TABLE workflow_errors (
  id SERIAL PRIMARY KEY,
  workflow_id VARCHAR(100),
  workflow_name VARCHAR(255),
  execution_id VARCHAR(100),
  error_message TEXT,
  error_stack TEXT,
  occurred_at TIMESTAMP DEFAULT NOW(),
  resolved BOOLEAN DEFAULT FALSE,
  resolved_at TIMESTAMP,
  notes TEXT
);
```

---

## Part 4: Structured Logging

### Logging Pattern

Add a Code node at key points in your workflow to emit structured log events:

```javascript
// Structured log emitter
function log(level, event, data) {
  console.log(JSON.stringify({
    timestamp: new Date().toISOString(),
    level,
    event,
    workflow: 'portfolio_advisor',
    execution_id: $execution.id,
    ...data
  }));
}

log('info', 'advisor_request_received', {
  investor_id: $json.investor_id,
  question_length: $json.question.length,
  question_category: 'health_check'
});

return [$input.item];
```

N8N writes `console.log` to its own log output. In production, ship N8N logs to a log aggregation service (Datadog, Loki, CloudWatch).

### Key Events to Log

| Event | Level | Data to Log |
|-------|-------|-------------|
| `request_received` | info | investor_id, question, timestamp |
| `tool_selected` | debug | tool_name, input_params |
| `tool_completed` | debug | tool_name, duration_ms, response_size |
| `ai_response_generated` | info | tools_used_count, response_length, duration_ms |
| `approval_sent` | info | request_id, reviewer_email, expires_at |
| `decision_received` | info | request_id, decision, reviewer, latency_hours |
| `error_occurred` | error | error_message, node_name, stack |
| `sla_breach` | warning | request_id, expected_response_hours, actual_hours |

---

## Part 5: SLA Tracking

### What is an SLA in This Context?

Service Level Agreement (SLA) targets for the portfolio advisory system:
- AI response time: < 30 seconds for 95% of queries
- Approval email sent: < 2 minutes of request receipt
- Compliance review completed: < 4 hours during business hours
- Redemption execution: < 1 business day of approval

### SLA Monitoring Workflow

Build a separate scheduled workflow that runs hourly:

```
Schedule Trigger (every 1 hour)
    ↓
Postgres: SELECT pending approvals older than X hours
    ↓
IF: any SLA breach?
    ├── YES → Telegram alert + update breach log
    └── NO  → Log: all within SLA
```

**SLA Query:**
```sql
SELECT 
  request_id, investor_name, submitted_at, redemption_amount,
  EXTRACT(EPOCH FROM (NOW() - submitted_at)) / 3600 AS hours_pending
FROM pending_approvals
WHERE status = 'pending'
  AND submitted_at < NOW() - INTERVAL '4 hours'
ORDER BY submitted_at ASC;
```

---

## Part 6: Capstone — AI Portfolio Intelligence Platform

### System Architecture

The capstone integrates all 7 days into a cohesive platform:

```
                    ┌─────────────────────────────────┐
                    │   AI Portfolio Intelligence      │
                    │         Platform                 │
                    └─────────────────────────────────┘
                                   │
              ┌────────────────────┼────────────────────┐
              │                    │                    │
     ┌────────▼────────┐  ┌───────▼────────┐  ┌───────▼────────┐
     │ Data Ingestion   │  │  AI Advisory   │  │  Monitoring    │
     │ (Day 2)         │  │  (Day 5+6)     │  │  (Day 3+7)    │
     │                 │  │                │  │                │
     │ • CAS Upload    │  │ • Portfolio Q&A│  │ • Health Score │
     │ • Parse PDF     │  │ • Stress Tests │  │ • Daily Alerts │
     │ • Store Holdings│  │ • Tax Planning │  │ • SLA Tracking │
     └────────┬────────┘  └───────┬────────┘  └───────┬────────┘
              │                    │                    │
              └────────────────────┼────────────────────┘
                                   │
                    ┌──────────────▼──────────────┐
                    │    Alpha Signals (Day 4)     │
                    │                             │
                    │  • Institutional Scanner    │
                    │  • Fund-Stock Lookup        │
                    │  • Oversold + Fund Buy      │
                    └──────────────┬──────────────┘
                                   │
                    ┌──────────────▼──────────────┐
                    │   Compliance Gate (Day 6)   │
                    │                             │
                    │  • Human Approval           │
                    │  • Audit Trail              │
                    │  • Timeout Escalation       │
                    └──────────────┬──────────────┘
                                   │
                    ┌──────────────▼──────────────┐
                    │   Quality (Day 7)            │
                    │                             │
                    │  • Pipeline Evaluator       │
                    │  • Error Monitoring         │
                    │  • Structured Logging       │
                    └─────────────────────────────┘
```

### Capstone Deliverables

Every participant demos:

1. **Live CAS Parse**: upload a real or sample CAS PDF, show the parsed holdings and HTML report
2. **Portfolio Health Dashboard**: show today's health score, explain one flag
3. **Alpha Scanner**: show the current week's top 3 institutional alpha signals
4. **AI Advisor Demo**: ask a live question, walk through which tools the AI called
5. **Human Gate Demo**: trigger an approval workflow, show the email, click approve
6. **Evaluator Results**: show the last evaluation run — score, grade, category breakdown

### Integration Test: End-to-End Flow

Run this sequence to verify the full platform works:

```bash
# Step 1: Upload a CAS
curl -X POST http://localhost:5678/webhook/cas-upload \
  -F "file=@sample_cas.pdf"

# Step 2: Check portfolio health  
curl http://localhost:5678/webhook/portfolio-health

# Step 3: Get Alpha signals
curl http://localhost:5678/webhook/alpha-scanner

# Step 4: Ask the AI advisor
curl -X POST http://localhost:5678/webhook/portfolio-advisor \
  -H "Content-Type: application/json" \
  -d '{"question": "Based on my current holdings and the institutional alpha signals, what should I do?"}'

# Step 5: Request a redemption (triggers human gate)
curl -X POST http://localhost:5678/webhook/redemption-request \
  -H "Content-Type: application/json" \
  -d '{"investor_id": "test_investor", "investor_name": "Test User", "investor_email": "test@test.com", "redemption_amount": 100000, "urgency": "normal", "purpose": "goal_withdrawal"}'

# Step 6: Run pipeline evaluation
# (Trigger manually from N8N UI)
```

---

## Part 7: Exercises

### Exercise 1: Semantic Similarity Scoring

Instead of regex-based rubric scoring, add a semantic similarity step:
1. After the AI responds, send both the AI answer and the reference answer to OpenAI's embeddings API
2. Compute cosine similarity between the embeddings
3. Score: similarity > 0.85 = good match, > 0.70 = partial match, < 0.70 = poor match

### Exercise 2: Regression Test Schedule

Schedule the evaluator to run every Monday at 6:00 AM. Compare this week's scores to last week's:
- If average score drops by more than 10 points: send a "regression detected" alert
- If scores improve: send a "quality improvement" celebration message (optional)

### Exercise 3: Add a New Test Case

Add a test case for the Alpha Scanner integration:
```json
{
  "test_id": "TC006",
  "category": "alpha_signals",
  "question": "Are there any stocks that multiple mutual funds are buying that are currently oversold? Give me your top 3 picks.",
  "expected_tools": ["getPortfolioXray"],
  "rubric": {
    "mentions_specific_stocks": "[A-Z]{2,10}\\b",
    "mentions_rsi": "RSI|relative strength",
    "mentions_fund_count": "\\d+.*fund|fund.*\\d+",
    "mentions_oversold": "oversold|below.*40|RSI.*<.*40",
    "gives_reasons": "because|reason|since|as.*fund"
  }
}
```

---

## Day 7 Summary and Workshop Wrap-Up

Over the past seven days you have built:

| Day | Artifact |
|-----|---------|
| 1 | Scheme Explorer — search and NAV fetching |
| 2 | CAS Upload Pipeline — PDF parsing + HTML report + human gate |
| 3 | Portfolio Health Dashboard — scheduled Xray + DB storage + alerts |
| 4 | Institutional Alpha Scanner — cross-dataset join + fund-stock lookup |
| 5 | AI Portfolio Advisor — MCP-powered natural language Q&A |
| 6 | Compliance-Gated Redemption Advisor — human approval + audit trail |
| 7 | Pipeline Evaluator — rubric scoring + structured logging + SLA tracking |

### Key Takeaways

1. **N8N is powerful for financial workflows** — the combination of HTTP nodes, AI Agent, Wait node, and Code node covers virtually every pattern in financial advisory automation
2. **Human gates are not optional in finance** — the Wait node makes them trivially easy to implement
3. **Evaluate before you deploy** — a pipeline that looks good in demos may hallucinate in production
4. **Institutional data is public and actionable** — AMC portfolio disclosures are free alpha that most retail investors ignore
5. **MCP transforms AI agents** — instead of writing tool call logic, you expose APIs as MCP tools and the AI routes itself

### Next Steps

- Deploy N8N to a VPS or cloud provider with Postgres backend
- Set up Nginx reverse proxy + SSL for the webhook URLs
- Implement investor authentication before exposing the advisor webhook
- Add more test cases to the evaluator (target 20+ for production)
- Integrate WhatsApp Business API for advisor notifications
- Build a React/Next.js frontend that calls your N8N webhooks

**Day 7 workflow to import:** `workflows/day7_pipeline_evaluator.json`

Congratulations on completing the N8N + MF Engine API One-Week Workshop!
