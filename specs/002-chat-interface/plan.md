# Implementation Plan: Chat Interface as the Main User Experience

**Branch**: `002-chat-interface` | **Date**: 2026-05-30 | **Spec**: [spec.md](./spec.md)

**Input**: Feature specification from `/specs/002-chat-interface/spec.md`

## Summary

Replace the fixed card-based walkthrough of feature `001-consumption-decision-agent` with a **conversational chat interface** as the primary user surface, while reusing `001`'s decision engine, mock MCPs, schemas, and event log unchanged. The user types natural-language messages; an LLM intent layer classifies each message (consumption request / approval-or-decline / follow-up question / feedback / smalltalk) and, for consumption intents, extracts a structured `ConsumptionRequest` that is funnelled into the existing `decisionEngine.compose()` pipeline. Results — profile, goal, authorization, quote+market panels, decision card, execution trace, feedback acknowledgement — are rendered as **inline generative-UI cards** inside the chat thread. The technical approach is to add a chat surface to the existing Next.js 16 app using the Vercel AI SDK 5 UI layer (`useChat` + streaming + tool/data parts mapped to the already-built `001` card components), an in-memory conversation store mirroring `001`'s session stores, and a deterministic mapping from interpreted approval messages onto `001`'s existing `ask` pending-store transitions so the **Decision Engine remains the sole approval/decision authority** (Constitution Principle I).

The LLM's expanded role here is **interpretation only** — classifying intent and extracting request fields and approval signals. It never composes the allow/ask/deny verdict and never triggers execution directly; every interpreted artifact is Zod-validated and routed through the existing `001` modules.

## Technical Context

**Language/Version**: TypeScript 5.6+, Node.js 22 LTS (unchanged from `001`)

**Primary Dependencies**:
- Next.js 16 (App Router, Server Actions, Route Handlers) — reused from `001`
- React 19
- Vercel AI SDK 5 — **UI layer added** this feature: `@ai-sdk/react` `useChat` for the streaming thread, `streamText`/`generateObject` for the intent + reply server route, tool/data message parts for inline generative-UI cards
- AI Gateway (Vercel) for provider routing — same single `AI_GATEWAY_API_KEY`
- shadcn/ui + Tailwind CSS 4 — reuse `001`'s card components (Profile, Goal, AuthSetup, QuoteCard, DecisionCard, ExecutionTrace, FeedbackForm) as inline chat cards; add chat-shell primitives (message list, composer, bubbles)
- Zod — schema validation for the new `MessageIntent` extraction and reuse of all `001` entity schemas (Principle IV)
- Vitest for unit/contract tests; Playwright for one chat happy-path E2E

**Storage**:
- Reuse `001`'s JSON fixtures under `data/mocks/` and in-memory session stores unchanged
- **New** in-memory `Conversation`/`Message` store (`lib/store/memory.ts` extension) keyed by `userId`; restored on reload for the demo user (FR-015). Reset per process; no DB
- `ask` lifecycle continues to live in `001`'s decision pending-store (15-min TTL); chat cards read their state from it (FR-010)

**Testing**:
- Vitest contract test for the new **intent-extraction** module: NL message → `MessageIntent` (Zod-valid), covering consumption / approval / decline / follow-up / feedback / smalltalk classes and missing-field detection (FR-004, FR-005, FR-006)
- Vitest unit test: deterministic mapping of an interpreted approve/decline/ambiguous signal onto the existing `ask` pending-store transition, asserting ambiguous → no execution (FR-009, FR-010, FR-011)
- Vitest unit test: conversation ordering preserved when a message arrives mid-decision (FR-016)
- Reuse `001`'s decision-matrix, hard-rule, and execution contract tests unchanged (this feature must not alter them — asserted by leaving those suites green)
- Deterministic mock LLM fixtures for intent classification in CI; live LLM only in `DEMO_LIVE_LLM=true`
- Playwright E2E: type NL hotel request → decision card renders inline → (ask) approve inline → execution card renders → feedback in-thread

**Target Platform**: Web (evergreen browsers). Deployed on Vercel or `next start` locally. No mobile-native, no voice/attachments (Assumptions).

**Project Type**: Web application — single Next.js project (frontend + route handlers co-located), continuing `001`'s structure.

**Performance Goals**:
- SC-002: decision card posted within 10 seconds of a well-formed request (LLM intent extraction + existing decision flow; rule path still sub-100ms)
- Streaming first-token/typing feedback under ~1s so the thread feels responsive
- SC-008: full demo narratable as one conversation in under 5 minutes

**Constraints** (Constitution — see Constitution Check):
- Decision Engine owns the final allow/ask/deny verdict; LLM intent layer only *interprets* input, never decides (Principle I)
- Interpreted approvals map deterministically onto the existing `ask` pending-store transition; ambiguous replies never execute (Principle I; FR-011)
- Hard rules still evaluated before AI fit check inside the reused `001` pipeline (Principle II)
- Order placement still uses only the official quote via reused `001` executor (Principle III)
- The new `MessageIntent` and extracted `ConsumptionRequest` are Zod-validated before crossing into the decision pipeline (Principle IV)
- Every decision/execution/expiry/feedback event triggered through the chat emits the same `001` audit events; **no new event types** are introduced (Principle V; FR-018)

**Scale/Scope**:
- Single demo user (`user_001`), single Japan-trip scenario, single active conversation (Assumptions)
- One concurrent `ask` per user; per-request serial evaluation (reused from `001`)
- ~4 new chat-shell components + reuse of 7 existing `001` cards as inline parts
- 1 new chat/intent route handler; 1 new conversation store; 1 new intent module

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Principle | Check | Status |
|---|---|---|
| I. Decision Engine Authority | The new LLM intent layer only **classifies and extracts** (`MessageIntent`). It produces a `ConsumptionRequest` and an advisory approve/decline/ambiguous **signal** — never an allow/ask/deny verdict. The verdict is still composed exclusively by `001`'s `decisionEngine.compose()`. Typed approvals invoke the **same** deterministic `decision.approveAsk()` / `decision.declineAsk()` transitions the button invokes; an ambiguous signal re-prompts and never executes (FR-011). The executor is still reached only from inside the decision flow. | ✅ PASS |
| II. Authorization Boundary Enforcement | Unchanged. Every chat-originated `ConsumptionRequest` enters `001`'s pipeline where `authorizationEngine.evaluate()` runs first and a hard-rule failure short-circuits to `deny` before any fit check. The chat layer adds no path that bypasses authorization. | ✅ PASS |
| III. Official Quote Is The Only Executable Quote | Unchanged. Inline approval delegates to `001`'s `execution.simulate()`, which reads only `officialQuote`. The chat renders the market panel for *reference/explanation only*; follow-up answers (FR-012) cite market data but never feed it to execution. | ✅ PASS |
| IV. Structured, Schema-First Module Contracts | New `MessageIntent` Zod schema added at `lib/schemas/message-intent.ts`; `Conversation`/`Message` schemas at `lib/schemas/conversation.ts`. The LLM intent output passes `safeParse` before the extracted `ConsumptionRequest` (validated against `001`'s existing schema) crosses into the decision pipeline. Inline cards consume the existing `001` entity schemas as data. | ✅ PASS |
| V. Mock-First Demo Scope & Auditable Event Trail | No new external integrations — only an added presentation/interpretation layer over the existing mocks. Every chat-triggered decision/execution/expiry/feedback emits the **same** canonical `001` events (`decision.made`, `order.created`, `payment.simulated`, `decision.expired`, `feedback.received`, `context.updated`). No new event types are added (FR-018). | ✅ PASS |

No violations. Complexity Tracking table not needed.

**Note on expanded LLM role**: This feature gives the LLM a new *input-interpretation* responsibility (intent classification + field/approval extraction). This is explicitly advisory under Principle I — interpreted output is Zod-coerced and funnelled through the deterministic decision and pending-store transitions. A contract test asserts the intent layer can never emit a verdict and that ambiguous approvals do not execute, keeping the NON-NEGOTIABLE boundary inspectable.

## Project Structure

### Documentation (this feature)

```text
specs/002-chat-interface/
├── plan.md              # This file
├── spec.md              # Feature spec
├── research.md          # Phase 0 output
├── data-model.md        # Phase 1 output
├── quickstart.md        # Phase 1 output
├── contracts/           # Phase 1 output
│   ├── chat-intent-contract.md   # NL message → MessageIntent; reply/card protocol
│   └── http-api.md               # POST /api/astra/chat streaming contract
├── checklists/
│   └── requirements.md  # Spec quality checklist (from /speckit-specify)
└── tasks.md             # Phase 2 output (NOT created here)
```

### Source Code (repository root)

Additions/changes layered on top of `001`'s tree (unchanged `001` paths omitted for brevity):

```text
app/
├── page.tsx                          # CHANGED: primary surface becomes the chat thread (FR-001)
├── walkthrough/page.tsx              # MOVED: the former 001 single-page walkthrough, kept as secondary/fallback view (Assumptions)
├── api/
│   └── astra/
│       └── chat/route.ts             # NEW: streaming chat endpoint — intent extraction → reply + inline card parts
└── components/
    └── chat/                         # NEW chat-shell primitives
        ├── ChatThread.tsx            # ordered, scrollable message list (FR-002, FR-015)
        ├── ChatComposer.tsx          # typed-text input + send (FR-004)
        ├── MessageBubble.tsx         # user/agent turn rendering
        └── InlineCard.tsx            # maps a card part → existing 001 card component (FR-003)

lib/
├── modules/
│   └── chat/                         # NEW conversational layer
│       ├── intent.ts                 # LLM intent classification + field/approval extraction → MessageIntent (Zod-coerced)
│       ├── orchestrate.ts            # routes a MessageIntent: request→decision pipeline, approval→pending-store, follow-up→explain, feedback→001 feedback
│       ├── explain.ts                # grounded follow-up answers over a recorded DecisionResult context (FR-012, FR-013)
│       └── index.ts
├── schemas/
│   ├── message-intent.ts             # NEW (Principle IV)
│   └── conversation.ts               # NEW Conversation + Message + EmbeddedCard schemas
├── store/
│   └── memory.ts                     # EXTENDED: conversation/message store keyed by userId (FR-015)
└── llm/
    └── prompts/
        ├── intent-classification.md  # NEW prompt
        └── follow-up-explain.md      # NEW prompt (grounded, no new verdict)

tests/
├── contract/
│   └── chat-intent.contract.test.ts  # NEW: message → MessageIntent shape + class coverage
├── unit/
│   ├── intent-extraction.test.ts     # NEW: field extraction + missing-field detection (FR-005)
│   ├── approval-mapping.test.ts      # NEW: approve/decline/ambiguous → pending-store transition (FR-010, FR-011)
│   └── conversation-ordering.test.ts # NEW: ordering preserved mid-decision (FR-016)
└── e2e/
    └── chat-happy-path.spec.ts       # NEW Playwright: NL request → inline decision → approve → execution → feedback

# Unchanged from 001: lib/modules/{profile,goal,authorization,quote,fitcheck,decision,execution,feedback,events}/,
# all lib/schemas/* entity schemas, data/mocks/*, and the 001 contract/unit/e2e suites.
```

**Structure Decision**: Continue `001`'s single Next.js 16 monolithic web app. The chat is added as a new primary surface (`app/page.tsx`) plus a thin `lib/modules/chat/` layer that **orchestrates** existing modules; it introduces no parallel decision logic. The former walkthrough is preserved under `app/walkthrough/` as a secondary view so `001`'s acceptance demo remains runnable. All decision-critical code (authorization, fit check, decision composition, execution, events) is reused verbatim, keeping the constitution boundaries intact at the source-tree level.

## Complexity Tracking

> No constitution violations — table omitted.
