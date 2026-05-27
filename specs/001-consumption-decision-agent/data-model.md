# Data Model: Consumption Decision Agent

**Date**: 2026-05-27 · **Feature**: 001-consumption-decision-agent

All entities below are defined as **Zod schemas** at `lib/schemas/<entity-kebab>.ts` and exported as both runtime validators and inferred TypeScript types. Every module-boundary function (per `contracts/module-contracts.md`) validates its inputs and outputs against these schemas (Constitution Principle IV).

Conventions: `string` IDs use a `<entity>_<slug>_<NNN>` pattern (e.g., `req_japan_hotel_001`). All amounts are integers in minor currency units? — **No**, the design doc uses whole SGD numbers; for demo simplicity we use `number` (decimal SGD). Timestamps are ISO 8601 strings.

---

## 1. UserBehaviorProfile

LLM-derived snapshot of a user's consumption behavior plus a derived per-category auto-execution threshold.

| Field | Type | Required | Notes |
|---|---|---|---|
| `userId` | `string` | yes | e.g., `user_001` |
| `snapshotId` | `string` | yes | Stable for the lifetime of this snapshot (FR-001b) |
| `generatedAt` | `string` (ISO datetime) | yes | |
| `profileTags` | `string[]` | yes | e.g., `["travel-positive", "budget-aware"]` |
| `avatarState` | `{ mood: string; message: string }` | yes | One-line message |
| `dimensions` | `Record<string, "low" \| "medium" \| "high">` | yes | ≥4 entries; LLM-determined names (FR-001b) |
| `thresholds` | `Record<Category, number>` | yes | Per-category auto-execution threshold in SGD (FR-001a) |

**Validation rules**:
- `dimensions` must have ≥4 entries (`refine`).
- Every category in `thresholds` must be one of `Category` enum below.
- `thresholds[c] > 0` for every entry.

**State transitions**: A profile snapshot is immutable. Feedback can produce a *new* snapshot but never mutates an existing one.

---

## 2. FinancialGoalContext

User's budget/savings posture and the demo scenario's impact.

| Field | Type | Required | Notes |
|---|---|---|---|
| `userId` | `string` | yes | |
| `monthlyBudget` | `number` | yes | SGD |
| `availableTravelBudget` | `number` | yes | SGD |
| `longTermSavingsGoal` | `{ name: string; targetAmount: number; currentAmount: number }` | yes | |
| `goalImpact` | `string` | yes | Natural-language sentence about the scenario |

**Validation rules**: `currentAmount ≤ targetAmount`; all amounts ≥ 0.

---

## 3. AuthorizationPolicy

Hard-rule boundary for a category of consumption.

| Field | Type | Required | Notes |
|---|---|---|---|
| `authorizationId` | `string` | yes | e.g., `auth_japan_trip_001` |
| `userId` | `string` | yes | |
| `type` | `"standing" \| "temporary"` | yes | FR-005 |
| `category` | `Category` | yes | enum (see below) |
| `maxSingleAmount` | `number` | yes | SGD, > 0 |
| `maxTotalAmount` | `number` | yes | SGD, ≥ `maxSingleAmount` |
| `currency` | `"SGD"` | yes | locked to SGD for demo |
| `createdAt` | `string` (ISO datetime) | yes | counting-window start (FR-004a) |
| `validUntil` | `string` (ISO datetime) | yes | counting-window end (FR-004a) |
| `executionFrequency` | `{ maxCount: number }` | yes | window is implicit per FR-004a |
| `executionMode` | `"manual" \| "auto" \| "manual_or_auto_by_amount"` | yes | |

**Validation rules**:
- `validUntil > createdAt`.
- `executionFrequency.maxCount ≥ 1`.

**State transitions**: Authorizations are append-only for the demo. Expiry is implicit (`validUntil < now`) and computed at evaluation time; no separate `expired` flag.

---

## 4. ConsumptionRequest

A request submitted by a consumption agent on the user's behalf.

| Field | Type | Required | Notes |
|---|---|---|---|
| `requestId` | `string` | yes | |
| `userId` | `string` | yes | |
| `agentId` | `string` | yes | e.g., `travel_agent` |
| `authorizationId` | `string` | yes | Which policy this request runs against |
| `category` | `Category` | yes | must match authorization category |
| `purpose` | `string` | yes | one-line natural language |
| `amount` | `number` | yes | SGD, > 0 |
| `currency` | `"SGD"` | yes | |
| `submittedAt` | `string` (ISO datetime) | yes | Used for frequency-window check (FR-004a) |

---

## 5. QuoteResult

Pair of (a) the executable official quote and (b) the market-intelligence reference. **Order placement reads only `officialQuote`** (Principle III).

| Field | Type | Required | Notes |
|---|---|---|---|
| `requestId` | `string` | yes | back-reference |
| `officialQuote` | `OfficialQuote` (below) | yes | |
| `marketIntelligence` | `MarketIntelligence` \| `null` | yes | `null` allowed (Edge: "market intelligence unavailable") |

**`OfficialQuote`**:
| Field | Type | Notes |
|---|---|---|
| `quoteId` | `string` | |
| `source` | `string` | e.g., `mock_merchant_mcp` |
| `merchantName` | `string` | |
| `amount` | `number` | SGD |
| `currency` | `"SGD"` | |
| `isExecutable` | `boolean` | `false` → no `allow` is possible (Edge case) |

**`MarketIntelligence`**:
| Field | Type | Notes |
|---|---|---|
| `source` | `string` | e.g., `mock_consumer_agent_network` |
| `marketAverage` | `number` | SGD |
| `priceRange` | `{ low: number; high: number }` | `low ≤ high` |
| `signal` | `"within_market_range" \| "above_market" \| "below_market"` | |

---

## 6. AIFitCheckResult

LLM-produced fit verdict; **never** the final allow/ask/deny.

| Field | Type | Required | Notes |
|---|---|---|---|
| `requestId` | `string` | yes | |
| `fitCheck` | `"pass" \| "caution" \| "fail"` | yes | FR-009 |
| `softRisks` | `string[]` | yes | possibly empty |
| `explanation` | `string` | yes | user-facing natural language |

---

## 7. DecisionResult

The final, rule-engine-owned verdict (Principle I).

| Field | Type | Required | Notes |
|---|---|---|---|
| `decisionId` | `string` | yes | |
| `requestId` | `string` | yes | |
| `authorizationId` | `string` | yes | the policy consulted |
| `decision` | `"allow" \| "ask" \| "deny"` | yes | |
| `reason` | `string` | yes | references rule(s) or fit-check result |
| `requiresUserApproval` | `boolean` | yes | `true` when `decision = "ask"` |
| `nextAction` | `"execute_order" \| "request_user_approval" \| "show_denial"` | yes | |
| `composedAt` | `string` (ISO datetime) | yes | drives the 15-min `ask` expiry (FR-015) |
| `expiresAt` | `string` \| `null` | yes | `composedAt + 15min` when `decision = "ask"`, else `null` |
| `status` | `"active" \| "approved" \| "declined" \| "expired" \| "executed"` | yes | `ask` lifecycle |

**State transitions** (`status`):

```
active ─approve─▶ approved ─execute─▶ executed
   │
   ├─decline──▶ declined
   │
   └─15 min──▶ expired   (emits decision.expired event)
```

Non-`ask` decisions move directly to terminal states: `allow → executed`, `deny → declined`.

---

## 8. ExecutionResult

Simulated order + payment outcome. Only created when the decision is `allow` (or an approved `ask`).

| Field | Type | Required | Notes |
|---|---|---|---|
| `executionId` | `string` | yes | |
| `decisionId` | `string` | yes | back-reference |
| `quoteId` | `string` | yes | the `officialQuote.quoteId` actually transacted (Principle III) |
| `amount` | `number` | yes | MUST equal `officialQuote.amount` (contract test asserts) |
| `orderStatus` | `"confirmed" \| "failed"` | yes | |
| `paymentStatus` | `"simulated_paid" \| "simulated_failed"` | yes | |
| `merchantOrderId` | `string` | yes | `mock_order_…` |
| `walletPaymentId` | `string` | yes | `mock_wallet_payment_…` |
| `executedAt` | `string` (ISO datetime) | yes | |

---

## 9. FeedbackRecord

User's post-decision response and the signals extracted from it.

| Field | Type | Required | Notes |
|---|---|---|---|
| `feedbackId` | `string` | yes | |
| `decisionId` | `string` | yes | |
| `executionId` | `string` \| `null` | yes | `null` if there was no execution (denied/expired) |
| `reaction` | `"positive" \| "negative" \| "neutral"` | yes | |
| `userFeedback` | `string` | yes | free text, may be empty |
| `contextSignals` | `string[]` | yes | LLM-extracted (FR-018); ≥1 element required |
| `submittedAt` | `string` (ISO datetime) | yes | |

---

## 10. Event

Single entry in the audit log (FR-019, Principle V).

| Field | Type | Required | Notes |
|---|---|---|---|
| `id` | `string` | yes | |
| `ts` | `string` (ISO datetime) | yes | |
| `type` | `EventType` (enum below) | yes | |
| `correlationId` | `string` \| `null` | yes | usually `requestId` or `decisionId` |
| `payload` | `Record<string, unknown>` | yes | event-specific, schema-free for the demo |

**`EventType` enum** (closed set; any other emission is a compile-time error):

```
profile.generated
goal.context.loaded
authorization.created
official_quote.received
market_intelligence.received
ai_fit_check.completed
decision.made
decision.expired
order.created
payment.simulated
feedback.received
context.updated
```

---

## Shared types

### `Category` enum

Per the design doc §4.4:

```
Daily | Travel | Health | Learning | Lifestyle | Financial
```

A `ConsumptionRequest.category` MUST equal the `AuthorizationPolicy.category` it runs against — enforced as a hard-rule check.

---

## Entity relationships

```text
UserBehaviorProfile ─┐                       
                     ├─▶ DecisionEngine ─▶ DecisionResult ─┐
FinancialGoalContext ┤                                     ├─▶ ExecutionResult ─▶ FeedbackRecord
AuthorizationPolicy ◀┤  (selected by ConsumptionRequest)   │
ConsumptionRequest ──┤                                     │
QuoteResult ─────────┤                                     │
AIFitCheckResult ────┘                                     │
                                                           ▼
                                                    Event[] (audit log)
```

Every transition in the diagram appends to the Event log per Principle V.
