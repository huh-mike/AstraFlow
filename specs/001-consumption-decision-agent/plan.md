# Implementation Plan: User-Profile-Based Consumption Decision Agent

**Branch**: `001-consumption-decision-agent` | **Date**: 2026-05-27 | **Spec**: [spec.md](./spec.md)

**Input**: Feature specification from `/specs/001-consumption-decision-agent/spec.md`

## Summary

Deliver an end-to-end hackathon web demo that walks a single mock user through **authorization → decision → order placement** for a Japan-trip scenario (5–7 days, S$2,000–2,500 budget). The system composes a deterministic `allow` / `ask` / `deny` decision per request from five inputs — user profile, financial-goal context, authorization policy, official merchant quote, market-intelligence reference — with the LLM acting as advisory context (profile generation, fit-check, explanations) and a rule engine owning the final verdict. All external dependencies are mocked. The technical approach is a single Next.js 16 (App Router) web app: React Server Components / Server Actions for the UI and decision flow, Vercel AI SDK for the LLM-driven Profile and AI Fit Check modules, structured TypeScript types matching the spec's Key Entities, and an in-memory + JSON-fixture data layer for all mock sources. A single end-to-end API route (`POST /api/demo/japan-trip/decision-flow`) covers the demo flow; module APIs are reachable individually for debugging.

## Technical Context

**Language/Version**: TypeScript 5.6+, Node.js 22 LTS

**Primary Dependencies**:
- Next.js 16 (App Router, Server Actions, Cache Components — used selectively for static profile/goal panels)
- React 19
- Vercel AI SDK 5 (`ai`, provider via AI Gateway) for LLM-driven Profile Analysis and AI Fit Check
- AI Gateway (Vercel) for provider routing — single env-var `AI_GATEWAY_API_KEY`
- shadcn/ui + Tailwind CSS 4 for demo UI (radar chart via `recharts`)
- Zod for schema validation at module boundaries (Constitution Principle IV)
- Vitest for unit/contract tests; Playwright for one happy-path E2E

**Storage**:
- JSON fixtures under `data/mocks/` for user data, merchant quotes, market intelligence (read at startup)
- In-memory `Map`-backed stores for the demo session — authorizations created in-session, decisions, executions, feedback, audit events. Reset per process; no DB
- Pending `ask` decisions expire via a 15-minute TTL tracked in the same in-memory store (FR-015)

**Testing**:
- Vitest for module-level contract tests (one per module boundary)
- Vitest for decision-matrix table tests covering every branch of FR-012 (6 branches) plus hard-rule edge cases
- Playwright for one E2E happy-path: profile load → set authorization → submit request → `allow` → execution + feedback
- Mock LLM responses via deterministic fixtures for CI; live LLM only in interactive demo mode (`DEMO_LIVE_LLM=true`)

**Target Platform**: Web (modern evergreen browsers, Chrome/Safari/Firefox latest). Deployed on Vercel (or `next start` locally). No mobile-native.

**Project Type**: Web application (single Next.js project — frontend + API routes co-located, per design doc §3 "monolithic web app with API routes is also acceptable")

**Performance Goals**:
- SC-002: end-to-end decision in under 10 seconds per request (LLM round-trips dominate; rule engine sub-100ms)
- SC-001: 5-minute presenter walkthrough — UI route transitions under 500ms
- Rule-engine path (hard-rule deny) MUST return in <100ms (no LLM call)

**Constraints**:
- Mock-first per Constitution Principle V — no real bank/card/payment/merchant integrations
- Decision Engine owns final allow/ask/deny — LLM output never reaches the executor directly (Principle I)
- Hard rules evaluated before AI fit check; hard-rule fail short-circuits to `deny` (Principle II)
- Order placement uses only the official quote, never market-intelligence average (Principle III)
- All module boundaries exchange Zod-validated schemas matching spec Key Entities (Principle IV)
- Every transition emits an event into the audit log (Principle V; FR-019)

**Scale/Scope**:
- Single demo user (`user_001`), single scenario (Japan trip)
- One concurrent `ask` per user; per-request serial evaluation (Edge Cases)
- ~7 demo UI panels (FR-020) on a single scrollable page
- ~9 mock JSON fixtures

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Principle | Check | Status |
|---|---|---|
| I. Decision Engine Authority | All LLM output (profile traits, fit-check verdict, explanation) is routed through `decisionEngine.compose()` which applies the deterministic FR-012 matrix. The executor `execution.simulate()` is invoked ONLY from inside `decisionEngine` when the verdict is `allow`. LLM modules return data, never call the executor. | ✅ PASS |
| II. Authorization Boundary Enforcement | `authorizationEngine.evaluate()` runs first in the decision pipeline; any hard-rule failure short-circuits to `deny` and skips the AI fit check entirely. Hard rules are pure functions over `AuthorizationPolicy` + `ConsumptionRequest`. | ✅ PASS |
| III. Official Quote Is The Only Executable Quote | `execution.simulate(decision, quotes)` accepts the full `QuoteResult` but its order-creation reads only `quotes.officialQuote.{amount, quoteId}`. A Vitest contract test asserts the executor never reads `quotes.marketIntelligence`. | ✅ PASS |
| IV. Structured, Schema-First Module Contracts | Every Key Entity has a Zod schema in `lib/schemas/`. Module functions accept/return only those schemas, validated at the boundary. LLM JSON output passes through `safeParse` before crossing a boundary. | ✅ PASS |
| V. Mock-First Demo Scope & Auditable Event Trail | Merchant, Wallet, consumer-agent network, user bank/Apple-Pay data implemented as mocks behind module boundaries. Every flow step calls `events.emit()`; the 12 canonical events of FR-019 are unit-tested for emission. | ✅ PASS |

No violations. Complexity Tracking table not needed.

## Project Structure

### Documentation (this feature)

```text
specs/001-consumption-decision-agent/
├── plan.md              # This file
├── spec.md              # Feature spec (with Clarifications session)
├── research.md          # Phase 0 output
├── data-model.md        # Phase 1 output
├── quickstart.md        # Phase 1 output
├── contracts/           # Phase 1 output (module + HTTP contracts)
│   ├── module-contracts.md
│   └── http-api.md
├── checklists/
│   └── requirements.md  # Spec quality checklist (from /speckit-specify)
└── tasks.md             # Phase 2 output (NOT created here)
```

### Source Code (repository root)

```text
app/                                  # Next.js App Router
├── page.tsx                          # Demo walkthrough page (single page, panels in scroll order per FR-020)
├── layout.tsx
├── api/
│   ├── astra/
│   │   ├── profile/[userId]/route.ts
│   │   ├── profile/analyze/route.ts
│   │   ├── goals/[userId]/route.ts
│   │   ├── authorizations/route.ts
│   │   ├── quotes/route.ts
│   │   ├── fit-check/route.ts
│   │   ├── decision/route.ts
│   │   ├── execute/route.ts
│   │   ├── feedback/route.ts
│   │   └── events/route.ts           # Read-only audit log
│   └── demo/
│       └── japan-trip/decision-flow/route.ts  # End-to-end aggregator
└── components/                       # Profile, Goal, AuthSetup, QuoteCard, DecisionCard, ExecutionTrace, FeedbackForm, EventLog
    └── ...

lib/
├── modules/
│   ├── profile/                      # Profile Analysis (LLM-driven)
│   │   ├── analyze.ts                # LLM call + Zod coercion
│   │   ├── threshold.ts              # Per-category auto-execution threshold derivation (FR-001a)
│   │   └── index.ts
│   ├── goal/
│   │   └── context.ts                # FinancialGoalContext loader
│   ├── authorization/
│   │   ├── engine.ts                 # evaluate() — deterministic hard rules (Principle II)
│   │   ├── policies.ts               # in-memory store
│   │   └── frequency.ts              # FR-004a window-bound counting
│   ├── quote/
│   │   ├── merchant.ts               # mock Merchant MCP
│   │   ├── market.ts                 # mock consumer-agent network
│   │   └── aggregate.ts              # Returns QuoteResult
│   ├── fitcheck/
│   │   └── ai-fit-check.ts           # LLM call → pass/caution/fail (Zod-coerced)
│   ├── decision/
│   │   ├── compose.ts                # FR-012 deterministic 6-branch matrix
│   │   ├── pending.ts                # 15-min TTL store for `ask` (FR-015)
│   │   └── index.ts
│   ├── execution/
│   │   ├── merchant-order.ts         # mock order create
│   │   ├── wallet-payment.ts         # mock payment
│   │   └── simulate.ts               # uses ONLY officialQuote (Principle III)
│   ├── feedback/
│   │   ├── record.ts                 # FeedbackRecord persistence
│   │   └── extract-signals.ts        # LLM-assisted signal extraction (FR-018)
│   └── events/
│       └── log.ts                    # Audit event emitter (FR-019)
├── schemas/                          # Zod schemas (Principle IV)
│   ├── user-behavior-profile.ts
│   ├── financial-goal-context.ts
│   ├── authorization-policy.ts
│   ├── consumption-request.ts
│   ├── quote-result.ts
│   ├── ai-fit-check-result.ts
│   ├── decision-result.ts
│   ├── execution-result.ts
│   ├── feedback-record.ts
│   └── event.ts
├── llm/
│   ├── client.ts                     # AI SDK client via AI Gateway
│   └── prompts/
│       ├── profile-analysis.md
│       ├── fit-check.md
│       └── feedback-signals.md
└── store/
    └── memory.ts                     # In-memory backed stores (per process)

data/mocks/
├── users/user_001.json
├── transactions/user_001.json
├── financial-accounts/user_001.json
├── goals/user_001.json
├── feedback-history/user_001.json
├── merchant-catalog/japan-trip.json
└── market-index/japan-trip.json

tests/
├── contract/                         # one per module boundary
│   ├── profile.contract.test.ts
│   ├── authorization.contract.test.ts
│   ├── quote.contract.test.ts
│   ├── fitcheck.contract.test.ts
│   ├── decision.contract.test.ts
│   ├── execution.contract.test.ts
│   └── feedback.contract.test.ts
├── unit/
│   ├── authorization-hard-rules.test.ts   # category, expiry, amount, frequency
│   ├── decision-matrix.test.ts            # 6 branches of FR-012
│   ├── frequency-window.test.ts           # FR-004a
│   ├── ask-expiry.test.ts                 # FR-015 15-min TTL
│   └── threshold-derivation.test.ts       # FR-001a
└── e2e/
    └── japan-trip-happy-path.spec.ts      # Playwright

CLAUDE.md                              # Updated to reference this plan
```

**Structure Decision**: Single Next.js 16 monolithic web app. The design doc explicitly permits this; a single app keeps the demo deployable to Vercel with one command. Module code lives under `lib/modules/<module>/` so the boundaries from the design doc are preserved at the source-tree level even without a separate `apps/api` package. UI components for the 7 panels in FR-020 live under `app/components/`.

## Complexity Tracking

> No constitution violations — table omitted.
