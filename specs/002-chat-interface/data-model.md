# Data Model: Chat Interface

**Date**: 2026-05-30 · **Feature**: 002-chat-interface

This feature adds **three** new entities for the conversational layer. All `001` entities (`UserBehaviorProfile`, `FinancialGoalContext`, `AuthorizationPolicy`, `ConsumptionRequest`, `QuoteResult`, `AIFitCheckResult`, `DecisionResult`, `ExecutionResult`, `FeedbackRecord`, `Event`) are **reused unchanged** and referenced by ID.

New entities are defined as Zod schemas at `lib/schemas/<entity-kebab>.ts` and validated at every module boundary (Constitution Principle IV). Conventions follow `001`: `<entity>_<slug>_<NNN>` IDs, ISO 8601 timestamps, SGD amounts as decimal `number`.

---

## 1. Conversation

An ordered session of interaction for one user. One active conversation per demo user.

| Field | Type | Required | Notes |
|---|---|---|---|
| `conversationId` | `string` | yes | e.g., `conv_user_001_001` |
| `userId` | `string` | yes | owner; `user_001` for the demo |
| `createdAt` | `string` (ISO datetime) | yes | |
| `messageIds` | `string[]` | yes | ordered; references `Message.messageId` in arrival order (FR-002, FR-015) |

**Validation rules**: `messageIds` preserves insertion order (append-only). No reordering or deletion in v1.

**State transitions**: Append-only. Reload restores the same `messageIds` order (FR-015).

---

## 2. Message

A single turn in a conversation. Either a user turn (typed text) or an agent turn (text + zero or more inline cards).

| Field | Type | Required | Notes |
|---|---|---|---|
| `messageId` | `string` | yes | e.g., `msg_user_001_007` |
| `conversationId` | `string` | yes | back-reference |
| `role` | `"user" \| "agent"` | yes | |
| `text` | `string` | yes | may be empty for a card-only agent turn |
| `cards` | `EmbeddedCard[]` | yes | possibly empty; only agent turns carry cards (FR-003) |
| `intentId` | `string \| null` | yes | for user turns, links to the `MessageIntent` interpreted from it; `null` for agent turns |
| `createdAt` | `string` (ISO datetime) | yes | arrival timestamp; drives ordering (FR-016) |

**Validation rules**:
- A `user` message MUST have `cards: []`.
- An `agent` message MAY have `cards`.
- Ordering is by `createdAt` / insertion; a message created while a decision is in flight keeps its arrival position (FR-016).

---

## 3. EmbeddedCard

A typed reference to a `001` artifact rendered inline inside an agent message, plus the display state needed to render it (FR-003). The card holds the **entity id**, not a copy of the entity — the renderer reads the live entity from the `001` store.

| Field | Type | Required | Notes |
|---|---|---|---|
| `cardId` | `string` | yes | |
| `type` | `CardType` (enum below) | yes | which `001` component to render |
| `entityId` | `string` | yes | id of the referenced `001` entity (e.g., `decisionId`, `requestId`, `snapshotId`) |
| `displayState` | `ApprovalDisplayState \| null` | yes | only for `type = "decision"` approval cards; `null` otherwise |

**`CardType` enum** (closed set; maps 1:1 to existing `001` components):

```
profile | goal | authorization | quote | decision | execution | feedback_ack
```

**`ApprovalDisplayState`** (for `ask` decision cards only — mirrors `DecisionResult.status`):

```
pending | approved | declined | expired
```

**Validation rules**:
- `displayState` is non-null **iff** `type = "decision"` and the referenced decision is/was an `ask`.
- `displayState` is derived from the live `DecisionResult.status` in `001`'s pending-store — the card never owns the source of truth (Principle I).

---

## 4. MessageIntent

The interpreted meaning of a **user** message, produced by the LLM intent layer and Zod-coerced before use. Never contains a verdict (Principle I).

| Field | Type | Required | Notes |
|---|---|---|---|
| `intentId` | `string` | yes | |
| `messageId` | `string` | yes | the user message it interprets |
| `intentType` | `IntentType` (enum below) | yes | classification |
| `request` | `ExtractedRequest \| null` | yes | non-null only for `consumption_request` |
| `approvalSignal` | `"approve" \| "decline" \| "ambiguous" \| null` | yes | non-null only for `approval`/`decline` intents (FR-009–FR-011) |
| `targetDecisionId` | `string \| null` | yes | for approval/decline/follow_up, the decision being acted on |
| `confidence` | `number` | yes | 0–1; low confidence on approval ⇒ treat as `ambiguous` |

**`IntentType` enum**:

```
consumption_request | approval | decline | follow_up | feedback | smalltalk
```

**`ExtractedRequest`** (fields extracted for a consumption intent — a *partial* `ConsumptionRequest` plus gaps):

| Field | Type | Notes |
|---|---|---|
| `category` | `Category \| null` | `001`'s `Category` enum; `null` if not stated |
| `purpose` | `string \| null` | one-line NL |
| `amount` | `number \| null` | SGD |
| `currency` | `"SGD" \| null` | defaults to `SGD` for the demo |
| `missingFields` | `string[]` | non-empty ⇒ agent must ask a clarifying question (FR-005) |

**Validation rules**:
- `intentType = "consumption_request"` ⇒ `request` non-null; if `request.missingFields` is non-empty, **no** `ConsumptionRequest` is built and **no** decision runs (FR-005).
- `intentType in {approval, decline}` ⇒ `approvalSignal` non-null and `targetDecisionId` non-null. `approvalSignal = "approve"` triggers `decision.approveAsk(targetDecisionId)`; `"ambiguous"` ⇒ re-prompt, no execution (FR-011).
- `intentType = "smalltalk"` ⇒ `request`, `approvalSignal`, `targetDecisionId` all `null`; agent replies conversationally and emits no decision/order (FR-006).
- The intent layer can **never** set an allow/ask/deny field — there is none on this schema (Principle I; asserted by contract test).

**Promotion to a `001` `ConsumptionRequest`**: when `missingFields` is empty, `orchestrate.ts` builds a full `ConsumptionRequest` (assigning `requestId`, `userId`, `agentId = "chat_user"`, and resolving `authorizationId` from the active trip authorization) and validates it against `001`'s existing `ConsumptionRequest` schema before entering the decision pipeline.

---

## Entity relationships

```text
Conversation 1──* Message 1──* EmbeddedCard ──refs──▶ 001 entities
                     │                                  (Profile/Goal/Auth/
                     │ (user turn)                        Quote/Decision/
                     ▼                                     Execution/Feedback)
                MessageIntent ──promote──▶ ConsumptionRequest ──▶ 001 Decision Pipeline
                     │  approve/decline                              │
                     └──────────────▶ decision.approveAsk/declineAsk ┘
                                                                     │
                                                            (emits 001 Events only)
```

- The chat layer **owns** `Conversation`, `Message`, `EmbeddedCard`, `MessageIntent`.
- It **reads/triggers** but never redefines `001` entities; all audit events are `001`'s existing `EventType` set — **no new event types** (FR-018, Principle V).
