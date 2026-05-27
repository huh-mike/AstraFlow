# Module Contracts

**Date**: 2026-05-27 · **Feature**: 001-consumption-decision-agent

Each module exposes a typed function signature. All inputs and outputs are Zod-validated at the boundary (Constitution Principle IV). LLM-producing modules `safeParse` the model's JSON output before returning.

Notation: `T` = Zod schema in `lib/schemas/`. `Promise<T>` = async returning a validated `T`.

---

## profile

```ts
// lib/modules/profile/analyze.ts
analyzeProfile(input: {
  userId: string;
  transactions: Transaction[];
  income: IncomeRecord[];
  expenses: ExpenseRecord[];
  budget: BudgetState;
  goals: FinancialGoal[];
  feedbackHistory: FeedbackRecord[];
}): Promise<UserBehaviorProfile>
```

Behavior:
- Calls LLM (`generateObject`) to produce `profileTags`, `avatarState`, `dimensions`.
- Calls `deriveThresholds(transactions, dimensions)` (see `threshold.ts`) to compute `thresholds`.
- Emits `profile.generated` event.
- **Postcondition**: `dimensions` has ≥4 entries; `thresholds` has one entry per `Category` present in `transactions` (plus `Travel` always present for the demo scenario).

```ts
// lib/modules/profile/threshold.ts
deriveThresholds(
  transactions: Transaction[],
  dimensions: Record<string, "low" | "medium" | "high">
): Record<Category, number>
```

Pure function. Algorithm per `research.md` R-2.

---

## goal

```ts
// lib/modules/goal/context.ts
loadFinancialGoalContext(userId: string, scenario: "japan_trip"): Promise<FinancialGoalContext>
```

Behavior:
- Reads the mock `goals/<userId>.json` fixture.
- Computes `availableTravelBudget` and `goalImpact` (deterministic, no LLM).
- Emits `goal.context.loaded` event.

---

## authorization

```ts
// lib/modules/authorization/policies.ts
createPolicy(input: Omit<AuthorizationPolicy, "authorizationId" | "createdAt">): Promise<AuthorizationPolicy>
getPolicy(authorizationId: string): Promise<AuthorizationPolicy | null>
listPolicies(userId: string): Promise<AuthorizationPolicy[]>
```

```ts
// lib/modules/authorization/engine.ts
evaluate(
  request: ConsumptionRequest,
  policy: AuthorizationPolicy,
  history: { executions: ExecutionResult[] }
): { ok: true } | { ok: false; failingRule: HardRule; reason: string }
```

`HardRule` = `"authorization_missing" | "expired" | "category_mismatch" | "over_single_amount" | "over_total_amount" | "frequency_exhausted"`.

Behavior:
- Pure function over `request`, `policy`, `history`.
- Frequency check uses `frequency.countWithinWindow(executions, policy)` (FR-004a).
- Never calls the LLM.

**Constitution gate (Principle II)**: `decisionEngine.compose()` MUST call `evaluate()` first and MUST NOT call the AI fit check when `evaluate()` returns `ok: false`. Enforced by a contract test that mocks both modules and asserts call order.

---

## quote

```ts
// lib/modules/quote/aggregate.ts
fetchQuotes(request: ConsumptionRequest): Promise<QuoteResult>
```

Behavior:
- Calls `merchant.fetchOfficialQuote(request)` and `market.fetchIntelligence(request)` in parallel.
- Emits `official_quote.received` and (if non-null) `market_intelligence.received` events.
- If the merchant call fails, returns a `QuoteResult` with `officialQuote.isExecutable = false`; the Decision Engine then refuses to emit `allow` (Edge case "official quote unavailable").

---

## fitcheck

```ts
// lib/modules/fitcheck/ai-fit-check.ts
runFitCheck(input: {
  request: ConsumptionRequest;
  profile: UserBehaviorProfile;
  goalContext: FinancialGoalContext;
  quotes: QuoteResult;
}): Promise<AIFitCheckResult>
```

Behavior:
- Calls LLM (`generateObject`) with `AIFitCheckResult` Zod schema.
- Emits `ai_fit_check.completed`.
- Returns `{ fitCheck: "pass" | "caution" | "fail", softRisks, explanation }`.
- **Never** computes a final decision (Principle I). Hard-coded prompt instruction reinforces this.

---

## decision

```ts
// lib/modules/decision/compose.ts
compose(input: {
  request: ConsumptionRequest;
  policy: AuthorizationPolicy;
  profile: UserBehaviorProfile;
  goalContext: FinancialGoalContext;
  history: { executions: ExecutionResult[] };
}): Promise<{ decision: DecisionResult; execution?: ExecutionResult }>
```

Behavior (the **only** function authorized to invoke `execution.simulate`):

1. Call `authorization.evaluate(request, policy, history)`.
   - If `ok: false` → emit `decision.made` with `decision = "deny"`, return. **Skip steps 2–4.**
2. Call `quote.fetchQuotes(request)`.
   - If `officialQuote.isExecutable === false` → compose with `decision = "deny"` (reason: "quote unavailable").
3. Call `fitcheck.runFitCheck(...)`.
4. Apply the FR-012 6-branch deterministic matrix:

| Hard | AI verdict | Amount vs `profile.thresholds[category]` | Decision |
|---|---|---|---|
| fail | – | – | `deny` |
| pass | `pass` | `< threshold` | `allow` |
| pass | `pass` | `≥ threshold` | `ask` |
| pass | `caution` | any | `ask` |
| pass | `fail` | `< threshold` | `ask with strong warning` |
| pass | `fail` | `≥ threshold` | `deny` |

5. Emit `decision.made`.
6. If `decision === "allow"` → call `execution.simulate(decisionResult, quotes.officialQuote)`, attach to return.
7. If `decision === "ask"` → register the decision in `decision/pending.ts` with `expiresAt = composedAt + 15min`.

```ts
// lib/modules/decision/pending.ts
register(decision: DecisionResult): void
approve(decisionId: string): Promise<ExecutionResult>     // moves status → approved → executed; calls execution.simulate
decline(decisionId: string): Promise<void>                // status → declined
sweepExpired(now: Date): Promise<DecisionResult[]>        // status → expired; emits decision.expired
```

---

## execution

```ts
// lib/modules/execution/simulate.ts
simulate(decision: DecisionResult, officialQuote: OfficialQuote): Promise<ExecutionResult>
```

Behavior:
- Calls `merchant-order.create(officialQuote)` → mock order.
- Calls `wallet-payment.pay(officialQuote.amount)` → mock payment.
- Emits `order.created`, `payment.simulated`.
- **Reads only `officialQuote`** — does NOT receive `marketIntelligence` and the function signature enforces this (Principle III). Contract test asserts `executionResult.amount === officialQuote.amount`.

---

## feedback

```ts
// lib/modules/feedback/record.ts
submit(input: {
  decisionId: string;
  executionId: string | null;
  reaction: "positive" | "negative" | "neutral";
  userFeedback: string;
}): Promise<FeedbackRecord>
```

Behavior:
- Calls `extractSignals(userFeedback, reaction, decision)` (LLM, `generateObject`) → ≥1 signal.
- Emits `feedback.received`, then `context.updated`.

---

## events

```ts
// lib/modules/events/log.ts
emit<T extends EventType>(type: T, payload: Record<string, unknown>, correlationId?: string): void
list(filter?: { correlationId?: string }): Event[]
```

Closed-enum `type` parameter (compile-time check). Pure append.
