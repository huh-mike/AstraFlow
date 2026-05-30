# Module Contract: Chat Intent & Orchestration

**Feature**: 002-chat-interface · **Date**: 2026-05-30

These are the internal module-boundary contracts for the conversational layer. All inputs/outputs are Zod-validated (Principle IV). The chat layer orchestrates the existing `001` modules and never re-implements decision logic.

---

## C1. `intent.classify(input) → MessageIntent`

`lib/modules/chat/intent.ts`

- **Input**:
  ```ts
  { messageId: string; userId: string; text: string; activeDecisionId: string | null }
  ```
  `activeDecisionId` is the currently-pending `ask` decision (if any) so approvals can be targeted.
- **Output**: `MessageIntent` (see data-model.md §4), Zod-coerced via `safeParse`.
- **Guarantees**:
  - Output contains **no** allow/ask/deny field (Principle I — schema has none).
  - `consumption_request` with any `missingFields` ⇒ caller MUST NOT build a `ConsumptionRequest` (FR-005).
  - `smalltalk` ⇒ `request`/`approvalSignal`/`targetDecisionId` are `null` (FR-006).
  - Low-confidence approval ⇒ `approvalSignal = "ambiguous"` (FR-011).
- **Errors**: LLM output failing `safeParse` ⇒ retry once, then fall back to `intentType = "smalltalk"` with a clarifying reply (never a fabricated decision).

---

## C2. `orchestrate.handle(intent) → AgentTurn`

`lib/modules/chat/orchestrate.ts` — routes a validated `MessageIntent`.

- **Output `AgentTurn`**:
  ```ts
  { text: string; cards: EmbeddedCard[]; events: EventType[] }
  ```
- **Routing table**:

  | `intentType` | Action | Cards emitted | 001 events |
  |---|---|---|---|
  | `consumption_request` (complete) | build + validate `ConsumptionRequest` → `001` decision pipeline | `decision` (+ `quote`) | `official_quote.received`, `market_intelligence.received`, `ai_fit_check.completed`, `decision.made`, (+ `order.created`,`payment.simulated` if `allow`) |
  | `consumption_request` (missing fields) | ask clarifying question | none | none |
  | `approval` (`approve`) | `decision.approveAsk(targetDecisionId)` → `001` execution | `execution`, updated `decision` card → `approved` | `order.created`, `payment.simulated` |
  | `approval`/`decline` (`ambiguous`) | re-prompt for explicit confirm/decline | none | none |
  | `decline` | `decision.declineAsk(targetDecisionId)` + record decline as feedback | `decision` card → `declined` | `feedback.received`, `context.updated` |
  | `follow_up` | `explain.answer(targetDecisionId, text)` | none (text only) | none |
  | `feedback` | `001` `feedback.record` + `extract-signals` | `feedback_ack` | `feedback.received`, `context.updated` |
  | `smalltalk` | conversational reply | none | none |

- **Guarantees**:
  - Execution is reached **only** via `001`'s decision/pending-store paths (Principle I, III).
  - No new `EventType` is ever emitted — only `001`'s closed set (Principle V, FR-018).
  - A `consumption_request` always traverses `authorizationEngine.evaluate()` first inside the `001` pipeline (Principle II).

---

## C3. `explain.answer(decisionId, question) → string`

`lib/modules/chat/explain.ts`

- **Input**: a recorded `decisionId` + the user's NL follow-up.
- **Context passed to LLM**: read-only `{ DecisionResult, AIFitCheckResult, QuoteResult, authorization outcome, profile/goal summary }` for that request.
- **Guarantees** (FR-012, FR-013):
  - Answer cites only the recorded context; introduces **no** new or contradictory verdict.
  - A hypothetical implying a different amount/authorization is framed as hypothetical or offered as a new request — the prior `DecisionResult` is never mutated.

---

## C4. Approval-mapping invariant (test target)

The function that maps an interpreted approval onto state MUST satisfy:

```
approvalSignal = "approve"   → decision.approveAsk(id)   (same path as the card button)
approvalSignal = "decline"   → decision.declineAsk(id)
approvalSignal = "ambiguous" → no state change, re-prompt
expired decision + "approve" → rejected with "window closed", offer re-submit
```

Asserted by `tests/unit/approval-mapping.test.ts` (FR-010, FR-011; Edge: approve-after-expiry).
