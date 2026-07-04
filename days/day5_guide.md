# Day 5: MCP Integration in N8N

## Learning Objectives

By the end of Day 5 you will be able to:
- Explain what MCP (Model Context Protocol) is and why SSE transport matters
- Connect MF Engine MCP servers as tools in N8N's AI Agent node
- Configure the AI Agent system prompt, tools, and memory
- Build a complete AI Portfolio Advisor that answers natural language questions
- Understand how the AI selects which MCP tool to call for a given query
- Demo the advisor with stress-test, tax-aware redemption, and SIP projection queries
- Route between multiple MCP servers based on query type

---

## Part 1: What is MCP?

### Model Context Protocol

MCP (Model Context Protocol) is an open standard developed by Anthropic that defines how AI models interact with external tools and data sources. Instead of writing custom API integration code for every tool, MCP provides a standardized protocol that allows:

1. **Tool Discovery**: the AI model can ask the MCP server "what tools do you have?" and get back a structured list with schemas
2. **Tool Invocation**: the AI calls a tool by name with structured parameters, and MCP handles the actual execution
3. **Result Handling**: the tool result is returned to the AI in a format it understands

Think of MCP as a "plugin system" for AI models. Instead of fine-tuning the model to know about your specific APIs, you expose those APIs as MCP tools and the model can use them in any conversation.

### Why MCP vs Direct HTTP Calls?

When you use a regular HTTP Request node in N8N, **you** decide which API to call based on your workflow logic. With MCP in an AI Agent, the **AI decides** which tool to call based on the user's question. This enables:

- Natural language queries ("What would happen to my portfolio if markets crash?")
- Multi-step reasoning (the AI calls getPortfolioXray first, then decides to call getPortfolioStressTest based on what it sees)
- Dynamic tool selection (for a SIP question, the AI picks getSipProjection; for a redemption question, it picks getSmartRedemptionPlan)

### SSE Transport

MCP supports multiple transports. The MF Engine uses **SSE (Server-Sent Events)** transport, which works as follows:

1. N8N opens an HTTP connection to the MCP server's SSE endpoint
2. The server streams events back (tool definitions, tool results) over this persistent connection
3. N8N sends tool call requests back via standard HTTP POST
4. This pattern works well in cloud/hosted environments and does not require WebSocket support

---

## Part 2: MCP Servers Available

### MF Engine MCP Endpoints

All MCP servers require the `x-api-key` header.

#### `/mcp/portfolio-analytics`

The most comprehensive MCP server. Tools:

| Tool | Description |
|------|-------------|
| `getPortfolioXray` | Full portfolio Xray — allocation, returns, concentration |
| `getPortfolioHealthScore` | 0–100 health score with component breakdown and flags |
| `getFundAnalytics` | Deep analytics for a specific fund (pass scheme_id) |
| `comparePortfolioFunds` | Side-by-side comparison of two or more funds |
| `getRedemptionRecommendations` | Smart redemption suggestions based on goal and tax |
| `getTaxAwareRedemptionPlan` | LTCG/STCG optimized redemption plan |
| `getFundVsIndexComparison` | Alpha generation vs benchmark for a fund |
| `getPortfolioAllocationAnalysis` | Target vs actual allocation gap analysis |
| `getSipProjection` | SIP wealth projection for a given fund and time horizon |
| `getSmartRedemptionPlan` | Complete redemption plan (amount, fund, timing, tax) |
| `getPortfolioFactorExposure` | Value/Growth/Quality/Momentum factor loadings |
| `getPortfolioStressTest` | Historical scenario stress test |
| `getTopAlternativeFunds` | Best alternative funds for a given scheme |
| `getCasImportHistory` | History of CAS imports for the investor |

#### `/mcp/portfolio-cas`

Tools for CAS-specific analysis:

| Tool | Description |
|------|-------------|
| `getPortfolioAnalytics` | CAS-derived portfolio analytics |
| `getSchemeResearch` | Research report for a specific scheme |
| `getBucketRecommendations` | Goal-bucket allocation recommendations |
| `getRebalancePlan` | Rebalancing plan to reach target allocation |
| `stageBucketPurchaseCart` | Stage a purchase cart for execution |

#### `/mcp/investor`

Investor-facing tools requiring investor JWT authentication.

#### `/mcp/partner`

Partner-facing tools for client management.

---

## Part 3: Connecting MCP in N8N AI Agent

### AI Agent Node Overview

The N8N AI Agent node (`@n8n/n8n-nodes-langchain.agent`) orchestrates an LLM (OpenAI, Anthropic, etc.) with a set of tools. The node:
1. Receives user input
2. Sends it to the LLM along with the tool definitions
3. The LLM decides which tool to call (or whether to respond directly)
4. N8N executes the tool call
5. The result is sent back to the LLM
6. Steps 3–5 repeat until the LLM gives a final answer
7. The final answer is emitted as the node's output

### Step-by-Step MCP Setup

#### Step 1: Add OpenAI Credential

1. Settings → Credentials → Add Credential → **OpenAI API**
2. Name: `OpenAI GPT`
3. API Key: your OpenAI key
4. Save

#### Step 2: Add the AI Agent Node

1. Add a new node: search for **AI Agent** (`@n8n/n8n-nodes-langchain.agent`)
2. In **Agent** settings:
   - Agent Type: **Tools Agent** (uses function calling)
   - Max Iterations: `10` (max tool calls per query)

#### Step 3: Add the LLM Sub-Node

1. Click the **+** on the Agent's **Model** connector
2. Select **OpenAI Chat Model** (`@n8n/n8n-nodes-langchain.lmChatOpenAi`)
3. Credential: select **OpenAI GPT**
4. Model: `gpt-4o` (recommended for tool use) or `gpt-4o-mini` (cheaper)
5. Temperature: `0.1` (low temperature for more deterministic, factual responses)
6. Max Tokens: `2000`

#### Step 4: Add MCP Client Tool for portfolio-analytics

1. Click the **+** on the Agent's **Tools** connector
2. Select **MCP Client** (`@n8n/n8n-nodes-langchain.mcpClientTool`) 
3. Name: `portfolio_analytics`
4. Connection Type: **SSE**
5. SSE URL: `https://app2.mfapis.club/mcp/portfolio-analytics`
6. Headers: `x-api-key: {{ $credentials.mfEngineApi.value }}`
   - Or hard-code: `x-api-key: your-api-key-here`
7. This single MCP node exposes ALL 14 tools from the portfolio-analytics server

#### Step 5: Add a Second MCP Client for portfolio-cas (optional)

Repeat Step 4 for `/mcp/portfolio-cas` to give the agent access to CAS-specific tools.

#### Step 6: Configure the System Prompt

In the AI Agent node, set the **System Message** (system prompt):

```
You are an expert mutual fund and investment portfolio advisor for Indian investors. 
You have access to real-time portfolio data via specialized tools.

IMPORTANT RULES:
1. Always call at least one data-fetching tool before giving investment recommendations — never answer from general knowledge alone.
2. When asked about portfolio health or overview, call getPortfolioHealthScore first, then getPortfolioXray for details.
3. When asked about redemption, call getSmartRedemptionPlan or getTaxAwareRedemptionPlan — specify whether the investor wants tax optimization.
4. When asked about SIP projections, call getSipProjection with the fund, monthly amount, and time horizon.
5. When asked about stress tests, call getPortfolioStressTest with the appropriate scenario.
6. Present numbers in Indian format (₹ symbol, lakhs/crores for large numbers).
7. Always mention XIRR when discussing returns (not just absolute returns).
8. Flag any concentration risk or red flags you see in the Xray data.
9. End every response with one specific, actionable recommendation.
10. Never recommend specific stock picks — focus on mutual fund strategies only.

TOOL SELECTION GUIDE:
- "how is my portfolio doing?" → getPortfolioHealthScore + getPortfolioXray
- "what happens if markets crash?" → getPortfolioStressTest (scenario: covid_2020 or gfc_2008)
- "which funds should I switch?" → comparePortfolioFunds + getTopAlternativeFunds
- "should I redeem?" → getSmartRedemptionPlan or getTaxAwareRedemptionPlan
- "SIP planning / wealth projection" → getSipProjection
- "factor analysis / style analysis" → getPortfolioFactorExposure
- "rebalancing" → getPortfolioAllocationAnalysis
```

---

## Part 4: Build — AI Portfolio Advisor

### Workflow Architecture

```
Webhook (POST /portfolio-advisor)
{
  "question": "Should I switch from HDFC Mid Cap to SBI Magnum Mid Cap?",
  "investor_id": "optional-for-multi-investor"
}
    ↓
Set Node: extract and validate question
    ↓
AI Agent Node
  ├── LLM: OpenAI GPT-4o
  ├── MCP Tools: portfolio-analytics (all 14 tools)
  └── MCP Tools: portfolio-cas (5 tools)
    ↓
Set Node: format response
    ↓
Respond to Webhook
```

### Step-by-Step Build

#### Step 1: Webhook Trigger

1. Add a **Webhook** node
2. Method: **POST**
3. Path: `portfolio-advisor`
4. Response Mode: `Using 'Respond to Webhook' Node`

#### Step 2: Set Node — Validate and Extract

1. Add a **Set** node named `Extract Question`
2. Fields:
   - `question`: `{{ $json.body.question }}`
   - `investor_id`: `{{ $json.body.investor_id || 'default' }}`
   - `received_at`: `{{ $now }}`

Add an IF node after this to check the question is not empty:
- Condition: `{{ $json.question }}` is not empty
- False branch: Respond to Webhook with 400 error

#### Step 3: AI Agent Node

1. Add the **AI Agent** node
2. Connect the Set node output to the Agent's main input
3. In the Agent's **Prompt** field:
   ```
   {{ $json.question }}
   
   Investor ID: {{ $json.investor_id }}
   Current Date: {{ $now.format('YYYY-MM-DD') }}
   ```
4. System Message: paste the system prompt from Part 3

5. Click **+** on Model connector → OpenAI Chat Model (GPT-4o)
6. Click **+** on Tools connector → MCP Client (portfolio-analytics server)
7. Click **+** on Tools connector again → MCP Client (portfolio-cas server)

#### Step 4: Set Node — Format Response

```javascript
// Mode: Run Once per Item
const agentOutput = $input.item.json.output; // AI Agent's text output
const toolsUsed = $input.item.json.intermediateSteps?.map(s => s.action?.tool).filter(Boolean) || [];

return {
  json: {
    answer: agentOutput,
    tools_called: [...new Set(toolsUsed)],
    question: $node['Extract Question'].json.question,
    responded_at: new Date().toISOString()
  }
};
```

#### Step 5: Respond to Webhook

Response body:
```json
{
  "question": "{{ $json.question }}",
  "answer": "{{ $json.answer }}",
  "tools_used": "{{ $json.tools_called }}",
  "timestamp": "{{ $json.responded_at }}"
}
```

### Testing the AI Advisor

Use curl to test different query types:

**Portfolio Health Query:**
```bash
curl -X POST http://localhost:5678/webhook-test/portfolio-advisor \
  -H "Content-Type: application/json" \
  -d '{"question": "Give me a complete health check of my portfolio. What are the main risks?"}'
```

**Stress Test Query:**
```bash
curl -X POST http://localhost:5678/webhook-test/portfolio-advisor \
  -H "Content-Type: application/json" \
  -d '{"question": "How would my portfolio have performed during the COVID crash of 2020? What was the drawdown?"}'
```

**Tax-Aware Redemption Query:**
```bash
curl -X POST http://localhost:5678/webhook-test/portfolio-advisor \
  -H "Content-Type: application/json" \
  -d '{"question": "I need ₹5 lakhs urgently. Which funds should I redeem first to minimize tax?"}'
```

**SIP Projection Query:**
```bash
curl -X POST http://localhost:5678/webhook-test/portfolio-advisor \
  -H "Content-Type: application/json" \
  -d '{"question": "If I start a SIP of ₹15,000 per month in HDFC Mid Cap for 20 years at 14% returns, how much will I have?"}'
```

**Fund Comparison Query:**
```bash
curl -X POST http://localhost:5678/webhook-test/portfolio-advisor \
  -H "Content-Type: application/json" \
  -d '{"question": "Should I switch from HDFC Mid Cap Opportunities to Nippon India Growth Fund? Compare their alpha generation."}'
```

---

## Part 5: How the AI Decides Which Tool to Call

### The ReAct Loop

The AI Agent uses a ReAct (Reasoning + Acting) loop:

```
User Query → AI Reasons → AI Decides on Tool → N8N Executes Tool → AI Observes Result → AI Reasons again → ...
```

For example, for the query "How bad would the COVID crash have been for my portfolio?":

```
Step 1 — Reasoning: "I need to know the portfolio composition before I can estimate stress impact."
Step 1 — Action: call getPortfolioXray

Step 2 — Observation: [Xray results with 75% equity, 15.8% small cap]
Step 2 — Reasoning: "Now I can run the stress test. The high small cap allocation will amplify the crash."
Step 2 — Action: call getPortfolioStressTest with scenario=covid_2020

Step 3 — Observation: [Stress results: -29.4% drawdown, 8 months recovery]
Step 3 — Reasoning: "I have enough data to provide a complete answer now."
Step 3 — Final Answer: "Based on your current allocation of 75% equity with 15.8% in small cap..."
```

### Influencing Tool Selection via System Prompt

The system prompt is the most powerful lever for guiding tool selection. Techniques:

1. **Explicit routing rules**: "When asked about stress tests, ALWAYS call getPortfolioStressTest FIRST" 
2. **Order specification**: "Call health score before Xray — health score is faster and tells you if Xray is needed"
3. **Negative rules**: "Never call more than 3 tools for a single query — be efficient"
4. **Fallback behavior**: "If a tool fails, tell the user what data you could not access and give a partial answer"

### Memory and Conversation History

The AI Agent node supports memory to maintain conversation context. Add a **Simple Memory** node:

1. Click **+** on Agent's **Memory** connector
2. Select **Window Buffer Memory** (`@n8n/n8n-nodes-langchain.memoryBufferWindow`)
3. Context Window: `10` (remember last 10 exchanges)
4. Session ID: `{{ $json.investor_id }}` (one memory per investor)

With memory enabled, the advisor can handle follow-up questions:
- Q1: "What is my portfolio health score?" → calls getPortfolioHealthScore
- Q2: "What about the debt portion?" → AI already knows the Xray from Q1 and can focus on debt

---

## Part 6: Multi-MCP Server Routing

### Adding All Four MCP Servers

You can connect all four MCP servers as tools in a single AI Agent:

```
AI Agent
├── LLM: GPT-4o
├── MCP: portfolio-analytics (14 tools)
├── MCP: portfolio-cas (5 tools)
├── MCP: investor (investor-specific tools)
└── MCP: partner (partner tools)
```

The AI will have access to all tools and will select the most appropriate one based on the query context.

### Routing Based on User Role

For multi-tenant setups (different users, different roles), build a router workflow:

```
Webhook
    ↓
Code Node: determine user role from JWT
    ↓
IF: role == "investor"
    ├── YES → AI Agent with investor + portfolio-analytics MCPs
    └── NO  → AI Agent with partner + portfolio-analytics MCPs
```

---

## Part 7: Demo Script

Walk through these demo queries in the workshop:

### Demo 1: Portfolio Health (2 minutes)
```
"Give me a full portfolio health report. Highlight any red flags."
```
Expected: AI calls getPortfolioHealthScore + getPortfolioXray, reports score, flags concentration risk, equity allocation.

### Demo 2: Stress Test (2 minutes)
```
"Run a 2008 financial crisis stress test on my portfolio. How long would it take to recover?"
```
Expected: AI calls getPortfolioStressTest with gfc_2008, reports drawdown %, trough value, estimated recovery months.

### Demo 3: Tax-Aware Redemption (3 minutes)
```
"I need ₹3 lakhs for my child's college fees next month. Which funds should I redeem to minimize LTCG tax?"
```
Expected: AI calls getTaxAwareRedemptionPlan with amount=300000, reports which units to redeem in which order.

### Demo 4: SIP Projection (2 minutes)
```
"If I start ₹10,000/month SIP in HDFC Flexi Cap for 15 years, assuming 12% annualized returns, what will I have?"
```
Expected: AI calls getSipProjection, returns projected value, wealth gain, year-by-year table.

### Demo 5: Follow-up with Memory (1 minute)
```
[After Demo 1] "What are the top 3 alternatives to my worst-performing fund?"
```
Expected: AI remembers the Xray from Demo 1, identifies bottom performer, calls getTopAlternativeFunds.

---

## Part 8: Exercises

### Exercise 1: Add a Confidence Field

Modify the output Set node to include a `confidence` field. Use the number of tool calls as a proxy:
- 3+ tools called: `high` confidence (AI did thorough research)
- 2 tools called: `medium` confidence
- 1 tool called: `low` confidence (minimal research)

### Exercise 2: Language Routing

Add a Code node before the AI Agent to detect if the user's question is in Hindi or Tamil. If so, add an instruction to the system prompt: "The user has asked in [language]. Respond in the same language."

### Exercise 3: Response Caching

Add a simple cache layer using N8N's Redis node:
1. Before the AI Agent, check Redis for a cached response for this question (use question hash as key)
2. If found and less than 1 hour old: return cached response directly (skip AI Agent)
3. If not found: run AI Agent, store response in Redis with 1-hour TTL

---

## Day 5 Summary

You learned how to:
- Understand MCP protocol and SSE transport mechanism
- Connect MCP servers to the N8N AI Agent node
- Configure system prompts for reliable tool selection
- Build a webhook-triggered AI Portfolio Advisor
- Enable conversation memory for follow-up queries
- Demo the advisor with real financial queries
- Route between multiple MCP servers based on user role

**Day 5 workflow to import:** `workflows/day5_ai_advisor.json`

Tomorrow in Day 6 you will add the human-in-the-loop approval gate to the AI advisor, making it compliant with financial regulations by requiring human review before delivering any redemption recommendations to clients.
