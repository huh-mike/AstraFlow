# User-Profile-Based Consumption Decision Agent Technical Design

This document records the technical implementation design for the current product direction. It does not replace the core product document.

## 1. Technical Goal

The technical implementation needs to support an end-to-end demo:

```text
Consumption authorization
  ↓
Consumption decision
  ↓
Consumption order placement
```

The product core is not product recommendation, and it is not simply a financial planning display. The technical core is enabling the system to output a clear decision based on user profile, financial goal, authorization boundary, official quote, and market reference:

- allow: the transaction is executed directly
- ask: the transaction requests user approval
- deny: the transaction is not executed

## 2. Demo Scope

Demo scenario:

```text
Japan trip for 5-7 days
Budget S$2000-2500
Covers pre-planning, order placement, and onsite consumption
```

The demo stage uses mock data for everything:

- mock user consumption data
- mock financial account data
- mock financial goals
- mock authorization policies
- mock Merchant MCP quote
- mock consumer agent network market intelligence
- mock Wallet MCP payment result

The demo stage does not integrate:

- real bank APIs
- real Apple Pay / card transaction history
- real Merchant MCP server
- real Wallet MCP server
- real agent discovery
- real payment

## 3. Recommended Technical Shape

Use a web demo architecture:

```text
Frontend
  ↓
Backend / API layer
  ↓
Mock data + decision services
  ↓
LLM service + rule engine
```

If based on the existing `finai` repo, this can map to:

```text
apps/web
  → demo UI, profile view, authorization setup, decision result, execution trace

apps/api
  → mock data API, profile analysis, authorization engine, decision engine, execution simulation
```

If building a lightweight demo from scratch, a monolithic web app with API routes is also acceptable. The key is not the stack itself, but clear interface boundaries and a clear decision flow.

## 4. System Modules

### 4.1 User Data Module

Provides demo user data.

Data includes:

- transaction records
- income records
- expense records
- budget state
- long-term financial goals
- user preference feedback
- historical consumption feedback

In the demo stage, all of this data comes from mock JSON.

### 4.2 Profile Analysis Module

Generates the user consumption behavior profile from user data.

Flow:

```text
mock user data
  ↓
LLM / agent analysis
  ↓
structured profile output
  ↓
profile tags + avatar state
```

The output needs to be structured JSON so later decision logic can reference it.

Profile dimensions will be refined later. For now, the system only needs to support a multi-dimensional structure that can later be used for radar chart display.

### 4.3 Financial Goal Module

Maintains the financial goal context in the demo.

Capabilities include:

- consumption budget planning
- long-term financial storage planning
- mock financial account state
- judging the impact of current consumption on future goals

It is not the main character of the demo. It provides context for consumption decisions.

### 4.4 Authorization Engine

Determines whether a consumption request is inside the user's authorization boundary.

Authorization types:

- standing authorization
- temporary authorization

Authorization dimensions:

- amount limits: short-term amount authorization, long-term amount authorization, per-transaction limit, total limit
- category scope: Daily, Travel, Health, Learning, Lifestyle, Financial
- execution frequency
- authorization time

The Authorization Engine only performs deterministic hard rule checks. It does not depend on the LLM.

### 4.5 Quote Aggregation Module

Gets executable quotes and market references.

The quote layer has two source types:

1. Official Quote Layer
   - Data source: mock Merchant MCP
   - Returns the current executable quote
   - Used for final order placement

2. Market Intelligence Layer
   - Data source: mock consumer agent network
   - Returns historical transaction prices, market averages, price ranges, and consumption feedback
   - Used only to judge whether the current quote is reasonable

Final order placement can only be based on the official quote. It cannot be based on historical prices from the consumer agent network.

### 4.6 AI Fit Check Module

Determines whether the consumption request fits the user profile and financial goal.

The AI fit check does not output a percentage score. It outputs three levels:

- pass
- caution
- fail

AI can judge:

- whether the current consumption matches the user's past spending behavior
- whether the current consumption matches the long-term financial goal
- whether the current quote is high compared with the market reference
- whether the current consumption has soft risks
- how to explain this decision to the user

AI does not directly decide whether money should be spent.

### 4.7 Decision Engine

Composes the final allow / ask / deny result.

Inputs:

- consumption request
- user profile
- financial goal context
- authorization policy
- official quote
- market intelligence
- AI fit check result

Decision rules:

```text
Hard rule fails
  → deny

Hard rule passes + AI fit check pass + low amount
  → allow

Hard rule passes + AI fit check caution
  → ask

Hard rule passes + AI fit check fail
  → deny or ask with strong warning
```

The Decision Engine is the final judge. The LLM is an analysis and explanation input, not the final authorization engine.

### 4.8 Execution Simulation Module

Simulates order placement and payment.

Information flow:

```text
Astra Flow / Agent Server
  ↓
mock Merchant MCP Server
  ↓
Create order / return order status
```

Fund flow:

```text
Astra Flow requests payment
  ↓
mock Wallet MCP Server executes payment
  ↓
mock Wallet MCP Server returns payment result
  ↓
mock Merchant MCP Server confirms order
```

No real payment happens in the demo. The system only generates an execution result and audit trail.

### 4.9 Feedback & Context Module

Handles the post-consumption feedback loop.

Flow:

```text
execution result
  ↓
decision explanation
  ↓
user feedback
  ↓
key decision signals
  ↓
user context update
  ↓
profile / goal adjustment
```

In the demo stage, user context can be updated as mock JSON or an event log.

## 5. Core Data Structures

### 5.1 UserBehaviorProfile

```json
{
  "userId": "user_001",
  "profileTags": ["travel-positive", "budget-aware", "prefers-convenience"],
  "avatarState": {
    "mood": "cautious",
    "message": "Your Japan trip is possible if onsite spending stays controlled."
  },
  "dimensions": {
    "budgetDiscipline": "medium",
    "travelPreference": "high",
    "priceSensitivity": "medium",
    "automationComfort": "medium"
  }
}
```

### 5.2 FinancialGoalContext

```json
{
  "monthlyBudget": 3200,
  "availableTravelBudget": 2500,
  "longTermSavingsGoal": {
    "name": "Emergency fund",
    "targetAmount": 10000,
    "currentAmount": 6200
  },
  "goalImpact": "Japan trip is affordable if total cost stays below S$2500."
}
```

### 5.3 AuthorizationPolicy

```json
{
  "authorizationId": "auth_japan_trip_001",
  "type": "temporary",
  "category": "Travel",
  "maxSingleAmount": 1200,
  "maxTotalAmount": 2500,
  "currency": "SGD",
  "validUntil": "2026-06-30",
  "executionFrequency": {
    "maxCount": 6,
    "window": "trip"
  },
  "executionMode": "manual_or_auto_by_amount"
}
```

### 5.4 ConsumptionRequest

```json
{
  "requestId": "req_japan_hotel_001",
  "userId": "user_001",
  "agentId": "travel_agent",
  "category": "Travel",
  "purpose": "Book hotel for Japan trip",
  "amount": 980,
  "currency": "SGD"
}
```

### 5.5 QuoteResult

```json
{
  "officialQuote": {
    "quoteId": "quote_hotel_001",
    "source": "mock_merchant_mcp",
    "merchantName": "Tokyo Stay Hotel",
    "amount": 980,
    "currency": "SGD",
    "isExecutable": true
  },
  "marketIntelligence": {
    "source": "mock_consumer_agent_network",
    "marketAverage": 910,
    "priceRange": {
      "low": 820,
      "high": 1100
    },
    "signal": "within_market_range"
  }
}
```

### 5.6 AIFitCheckResult

```json
{
  "fitCheck": "pass",
  "softRisks": [
    "Hotel cost is within the expected range for a 5-7 day Japan trip."
  ],
  "explanation": "This booking fits the user's Japan trip goal and stays within the authorized travel budget."
}
```

### 5.7 DecisionResult

```json
{
  "decision": "allow",
  "reason": "Hard rules passed, AI fit check passed, and the amount is below the auto-execution threshold.",
  "requiresUserApproval": false,
  "nextAction": "execute_order"
}
```

### 5.8 ExecutionResult

```json
{
  "executionId": "exec_hotel_001",
  "orderStatus": "confirmed",
  "paymentStatus": "simulated_paid",
  "merchantOrderId": "mock_order_001",
  "walletPaymentId": "mock_wallet_payment_001"
}
```

### 5.9 FeedbackRecord

```json
{
  "feedbackId": "feedback_001",
  "executionId": "exec_hotel_001",
  "userFeedback": "This matched my plan.",
  "contextSignals": [
    "user_accepts_mid-range_hotel_for_japan_trip",
    "travel_auto_execution_confidence_increased"
  ]
}
```

## 6. API Design

The demo can start with one end-to-end API, or it can be split into module APIs.

### 6.1 Single End-to-End API

```text
POST /api/demo/japan-trip/decision-flow
```

Input:

```json
{
  "userId": "user_001",
  "scenario": "japan_trip",
  "request": "Plan and book a Japan trip for 5-7 days within S$2000-2500."
}
```

Output:

```json
{
  "profile": {},
  "goalContext": {},
  "authorization": {},
  "quotes": {},
  "fitCheck": {},
  "decision": {},
  "execution": {},
  "feedbackPrompt": {}
}
```

### 6.2 Module APIs

```text
GET  /api/astra/profile/:userId
POST /api/astra/profile/analyze
GET  /api/astra/goals/:userId
POST /api/astra/authorizations
POST /api/astra/quotes
POST /api/astra/fit-check
POST /api/astra/decision
POST /api/astra/execute
POST /api/astra/feedback
```

The hackathon demo should prioritize getting the end-to-end flow working. Module APIs can be split gradually during implementation.

## 7. Demo Page Recommendation

The demo page should present the flow in this order:

1. User profile and avatar
2. Japan trip goal and budget range
3. User authorization setup
4. Official quote and market intelligence
5. Allow / ask / deny decision card
6. Order placement and payment simulation trace
7. Consumption feedback and user context update

The UI focus is not showing a recommendation list. The focus is showing how the system makes consumption decisions.

## 8. Event Log

The hackathon stage does not need Kafka. A simple event log can simulate the flow.

Events include:

```text
profile.generated
goal.context.loaded
authorization.created
official_quote.received
market_intelligence.received
ai_fit_check.completed
decision.made
order.created
payment.simulated
feedback.received
context.updated
```

These events are used to show the audit trail and consumption feedback loop.

## 9. LLM and Rule Engine Boundary

LLM is responsible for:

- analyzing user data
- generating profile tags and avatar state
- judging whether the consumption request fits the user profile and financial goal
- identifying soft risks
- generating natural-language explanations

The rule engine is responsible for:

- whether authorization exists
- whether authorization has expired
- whether the amount exceeds limits
- whether the category is out of scope
- whether execution frequency exceeds limits
- composing the final allow / ask / deny decision

Principle:

```text
LLM provides context and explanation.
Rule engine owns permission and final decision.
```

## 10. Hackathon Acceptance Criteria

The demo works if the following flow runs end to end:

1. Show mock user profile and avatar.
2. Show Japan trip for 5-7 days with S$2000-2500 budget goal.
3. User creates or views a Travel temporary authorization.
4. System gets the official MCP quote.
5. System gets the consumer agent network market reference.
6. System runs the hard rule check.
7. System runs the AI fit check.
8. System outputs allow / ask / deny.
9. If allow, simulate order placement and payment; if ask, show user confirmation; if deny, show the reason for non-execution.
10. System generates the consumption decision explanation.
11. After user feedback, system updates user context / profile signals.

## 11. Non-Goals

The demo stage does not include:

- real payment
- real bank data integration
- real Apple Pay transaction history reading
- real Merchant MCP integration
- real Wallet MCP integration
- real consumer agent network
- complex product recommendation algorithms
- complex financial planning productization

These can be future directions, but they are not the current demo core.
