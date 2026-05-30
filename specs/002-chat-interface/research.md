# Research: Chat Interface as the Main User Experience

**Date**: 2026-05-30 · **Feature**: 002-chat-interface

This feature adds a conversational layer over the existing `001-consumption-decision-agent`. No Technical Context entries were left as NEEDS CLARIFICATION — the stack is inherited from `001`. The decisions below resolve the *new* questions the chat surface introduces.

---

## R1. Chat UI + streaming approach

- **Decision**: Use the Vercel AI SDK 5 React UI layer (`@ai-sdk/react` `useChat`) for the thread, backed by a single streaming route handler (`POST /api/astra/chat`) that uses `streamText`. Inline cards are emitted as typed **message parts** (tool/data parts) and rendered by a client `InlineCard` switch that delegates to the existing `001` card components.
- **Rationale**: `001` already depends on AI SDK 5 and AI Gateway; reusing the same SDK avoids a second client/runtime. `useChat` gives ordered streaming messages, pending/typing state, and message-parts out of the box, which directly serve FR-002 (ordered thread) and FR-003 (inline generative-UI cards). Mapping parts → existing components means zero re-implementation of the profile/quote/decision cards.
- **Alternatives considered**:
  - *Hand-rolled WebSocket/SSE chat*: more control but re-implements ordering, streaming, and reconnection the SDK already provides; unjustified for a single-user demo.
  - *Plain request/response (no streaming)*: simpler, but loses the typing/streaming feel the success criteria value (SC-008 narration) and makes long LLM turns feel frozen.
  - *Vercel Chat SDK (multi-platform bot framework)*: aimed at Slack/Telegram/Teams bots, not an in-app web thread; wrong fit for an embedded demo UI.

## R2. Natural-language intent classification & field extraction

- **Decision**: A dedicated intent module (`lib/modules/chat/intent.ts`) calls the LLM with a constrained schema (`generateObject` against `MessageIntent` Zod schema) to classify each user message into one of `consumption_request | approval | decline | follow_up | feedback | smalltalk`, and for `consumption_request` extract `{ category, purpose, amount, currency }` plus a `missingFields[]` list. Output is `safeParse`d before use.
- **Rationale**: Structured output keeps the LLM's role to *interpretation* (Principle I) and produces a schema-validated object at the boundary (Principle IV). The explicit `missingFields[]` drives FR-005 (ask a clarifying question instead of guessing). Classifying smalltalk explicitly satisfies FR-006 (never fabricate a decision).
- **Alternatives considered**:
  - *Free-form parsing with regex/keyword rules*: brittle for natural phrasing ("around a grand", "book the Tokyo hotel"); the whole point of the chat UX is robust NL.
  - *Single mega-prompt that classifies AND decides*: violates Principle I (LLM would be deciding); rejected outright.
  - *Tool-calling where the model directly calls a `placeOrder` tool*: would let the model trigger execution; rejected. The model may only call an interpretation tool; execution stays behind the deterministic decision flow.

## R3. Mapping interpreted approvals onto the existing `ask` lifecycle

- **Decision**: A typed approval/decline is interpreted into an advisory signal, then mapped **deterministically** onto `001`'s existing decision pending-store transitions (`approveAsk(decisionId)` / `declineAsk(decisionId)`) — the same functions the on-card button invokes. Only an unambiguous `approval` signal triggers `approveAsk`; `ambiguous`/low-confidence triggers a re-prompt and never executes.
- **Rationale**: Keeps the Decision Engine / pending-store as the single approval authority (Principle I) while still letting users say "yes, go ahead". Reusing the button's code path means the button and the typed reply are provably equivalent. The ambiguous→re-prompt rule encodes FR-011 and the "maybe" edge case.
- **Alternatives considered**:
  - *Let the LLM emit the approval directly*: violates Principle I; the model would be authorizing execution.
  - *Buttons only, no typed approval*: simpler but breaks the "main UX is conversation" promise (FR-009 explicitly wants both).

## R4. Grounded follow-up explanations

- **Decision**: `lib/modules/chat/explain.ts` answers "why?"-style follow-ups using only the recorded `DecisionResult` + its linked `AIFitCheckResult`, `QuoteResult`, and authorization outcome for that request, passed to the LLM as read-only context with an instruction to explain — not re-decide. Hypotheticals that imply a different amount/authorization are answered as hypothetical or offered as a new request.
- **Rationale**: Satisfies FR-012/FR-013 and keeps explanations consistent with the actual verdict (Principle I — no new/contradictory verdict). Grounding on the stored decision context prevents the model from inventing rules.
- **Alternatives considered**:
  - *Re-run the decision pipeline for each follow-up*: wasteful and risks a different verdict if mock state drifted; rejected.
  - *Unconstrained chat memory*: model could contradict the recorded decision; rejected.

## R5. Conversation persistence & ordering

- **Decision**: Extend the existing in-memory session store with a `Conversation`/`Message` structure keyed by `userId`, appended in arrival order; the thread is restored for the demo user on reload. Messages arriving while a decision is in flight are appended in order; the in-flight decision posts its result when ready (FR-016).
- **Rationale**: Mirrors `001`'s in-memory, process-scoped store (no DB) — consistent with Mock-First scope (Principle V) and the single-user demo. Append-order storage gives FR-002/FR-015 ordering and reload restoration cheaply.
- **Alternatives considered**:
  - *Persistent DB (SQLite/Postgres)*: out of scope for the demo; `001` deliberately uses in-memory stores.
  - *Client-only state*: lost on reload, fails FR-015 restoration.

## R6. Relationship to the former walkthrough page

- **Decision**: The chat thread becomes the primary route (`app/page.tsx`); the former `001` single-page walkthrough is preserved at `app/walkthrough/` as a secondary/fallback view.
- **Rationale**: The spec makes chat the *main* UX (FR-001) but the Assumptions keep the walkthrough available; preserving it keeps `001`'s acceptance demo runnable and de-risks the change.
- **Alternatives considered**:
  - *Delete the walkthrough*: loses `001`'s demonstrable acceptance path and increases regression risk for no benefit.

---

## Summary of resolved unknowns

| Question | Resolution |
|---|---|
| How to render chat + inline cards | AI SDK 5 `useChat` + typed message parts → existing `001` card components (R1) |
| How NL becomes a structured request | Constrained `generateObject` → `MessageIntent` with `missingFields[]` (R2) |
| How typed approvals stay rule-owned | Interpreted signal → deterministic `approveAsk/declineAsk`; ambiguous → re-prompt (R3) |
| How follow-ups stay consistent | Grounded explanation over recorded decision context, no re-decide (R4) |
| How conversations persist | In-memory `Conversation` store keyed by user, append-ordered, reload-restored (R5) |
| Fate of the old walkthrough | Kept as secondary `app/walkthrough/` route (R6) |

All decisions preserve Constitution Principles I–V; the LLM's new role is strictly interpretation at a Zod-validated boundary.
