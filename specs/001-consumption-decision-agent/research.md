# Phase 0 Research: Consumption Decision Agent

**Date**: 2026-05-27 · **Feature**: 001-consumption-decision-agent

The Technical Context in `plan.md` had no remaining `NEEDS CLARIFICATION` items thanks to the design doc and the `/speckit-clarify` session. The five research entries below capture the engineering decisions that backstop those choices.

---

## R-1: Decision pipeline ordering (Authorization → AI fit check → Decision composition)

**Decision**: The decision pipeline runs strictly in three stages: (1) `authorizationEngine.evaluate(req, policy)` — pure-function hard rules; on failure, return `{decision: 'deny', reason: <rule>}` and skip stages 2–3; (2) `quoteAggregate.fetch(req)` + `aiFitCheck.evaluate(...)` — only invoked if stage 1 passes; (3) `decisionEngine.compose(...)` — applies the FR-012 6-branch matrix and emits `decision.made`.

**Rationale**:
- Constitution Principles I & II demand rule-engine authority and hard-rule precedence.
- Short-circuiting on hard-rule failure satisfies the <100 ms target for the deny-without-LLM path and SC-005 ("100% of rule violations → deny without AI fit check").
- The aggregator (`POST /api/demo/japan-trip/decision-flow`) is a thin orchestrator over these three calls; this lets the module-level APIs (`/api/astra/*`) reuse the same underlying functions for debugging without duplicating logic.

**Alternatives considered**:
- Running AI fit check in parallel with hard rules to save latency — rejected: violates Principle II (hard rules must complete first) and risks token spend on requests that will be denied anyway.
- Letting the AI fit check propose a decision and the rule engine "veto" — rejected: inverts authority, harder to audit.

---

## R-2: Profile-derived per-category auto-execution threshold (FR-001a)

**Decision**: The threshold is computed inside the Profile module after the LLM trait analysis. Algorithm: for each spending category present in the user's transaction history, compute the **median** historical transaction amount; then multiply by an automation-comfort multiplier ∈ [0.5, 1.5] derived from the LLM-determined trait map (specifically any dimension whose semantic label maps to "automation comfort" / "self-trust" — matched by case-insensitive substring on the LLM dimension names, with fallback constant 1.0 if no such dimension exists). Output: `thresholds: Record<Category, number>` on `UserBehaviorProfile`.

**Rationale**:
- Median (vs. mean) resists outliers from one-off large purchases.
- The multiplier turns the abstract LLM trait into a concrete number without re-prompting; cheap and deterministic.
- The fallback constant (1.0) ensures the system still works when the LLM produces dimension names unrelated to automation comfort — consistent with Q4's "LLM-determined dimensions, no fixed schema".
- The Decision Engine treats `thresholds[category]` as a black-box number, so swapping the derivation later is contained.

**Alternatives considered**:
- Asking the LLM to output the threshold directly — rejected: violates Principle I (LLM data must be coerced/transformed before becoming a decision parameter) and is less stable across runs.
- A global threshold per user (ignoring category) — rejected: FR-001a is explicit that the threshold is **per category**, and it makes the Japan-trip demo more interesting (small `Travel` items auto-pass, larger `Travel` items go to `ask`).

---

## R-3: 15-minute `ask` expiry implementation (FR-015)

**Decision**: Pending `ask` decisions are stored in an in-memory `Map<decisionId, PendingAsk>` keyed by decision ID. Each entry carries `createdAt`. Expiry is **lazy + opportunistic**:
1. On any read (`GET /api/astra/decision?id=…` or the next decision request from the same user), entries whose `createdAt + 15min < now` are evicted and a `decision.expired` event is emitted for each.
2. A `setInterval` sweep every 60s in the same Node process double-checks (covers idle UIs).
3. When a user explicitly approves or declines, the entry is removed and the appropriate event emitted (`order.created` + `payment.simulated` on approve; no event on decline beyond the existing `decision.made` update).

**Rationale**:
- Lazy eviction is simple, deterministic, and works in a single Node process (the demo target).
- The 60s sweep ensures the audit log gets `decision.expired` events even if no one reloads.
- No external scheduler/cron needed — keeps the demo single-process.

**Alternatives considered**:
- Vercel Cron Jobs — rejected: overkill for a single-process demo, adds infra complexity.
- Server-Sent Events / WebSockets to push expiry to the UI — rejected: the spec only requires the `pending → expired` state on next load (FR-015), not real-time push.

---

## R-4: LLM provider, prompt-structure, and JSON-output reliability

**Decision**: Use Vercel AI SDK 5 with `generateObject` (Zod schema as the second argument) for the two structured LLM calls — Profile Analysis and AI Fit Check — routed through Vercel AI Gateway. Default model `anthropic/claude-sonnet-4-6` (mapped via gateway); fallback `openai/gpt-4o-mini` if the gateway routes there. Prompts live in `lib/llm/prompts/*.md` and are loaded as plain strings.

**Rationale**:
- `generateObject` returns Zod-validated objects, eliminating the JSON-parsing failure mode and directly satisfying Constitution Principle IV at the LLM boundary.
- Vercel AI Gateway gives provider failover and a single env var (`AI_GATEWAY_API_KEY`), simplifying demo setup.
- Markdown-formatted prompt files keep prompt engineering legible and out of TS source.
- Claude Sonnet 4.6 is the cheapest model that reliably produces multi-dimensional profile JSON and reasoned fit-check verdicts in our pilot.

**Alternatives considered**:
- Direct provider SDKs (`@anthropic-ai/sdk`) — rejected: adds env-var/multi-provider complexity that AI Gateway already solves.
- `streamText` with manual JSON parsing — rejected: brittle, contradicts Principle IV.
- LangChain.js — rejected: overhead for two LLM calls.

> Note: AI SDK API surface (`generateObject` signature, AI Gateway provider syntax) is fluid — the implementing engineer MUST verify against https://sdk.vercel.ai/docs before coding. This research records only the architectural choice.

---

## R-5: Audit event log — store, ordering, and read API (FR-019, Principle V)

**Decision**: A single append-only in-memory `events[]` array (per process), each entry `{ id, ts, type, payload, correlationId }`. Modules call `events.emit(type, payload, correlationId)` where `correlationId` is the active `requestId` or `decisionId`. The log is exposed read-only via `GET /api/astra/events?correlationId=…` and rendered in the UI's Event Log panel. The 12 canonical event types (the 11 in FR-019 plus `decision.expired` from Q3) are enumerated in `lib/schemas/event.ts` as a Zod `enum`; any other emission rejected at compile time.

**Rationale**:
- Append-only array preserves causal order — critical for SC-003 ("all eleven canonical events appear in order").
- `correlationId` lets the UI filter the log per request, supporting the per-decision audit-trail use case.
- Enum-typed event names prevent silent typos that would break SC-003.
- In-memory is acceptable because the demo is single-process and a refresh resets state intentionally.

**Alternatives considered**:
- Persist to JSON file on disk — rejected: unneeded for a demo, complicates Vercel deploys.
- Use OpenTelemetry / a real tracer — rejected: out of scope for a hackathon demo per Principle V (the event log IS the observability for this stage).

---

## Open questions deferred to implementation

None blocking. Two notes for the implementer:

1. **LLM cost ceiling**: cap the demo to ~50 LLM calls per process restart; surface a warning banner in the UI if the running cost (rough estimate from token counts) exceeds a configurable threshold. Non-blocking, additive.
2. **Mock fixture realism**: the Japan-trip merchant catalog and market index should include at least 8 line items (hotel, flight, several meals, transit, attractions) so the demo can hit multiple categories and exercise more than one threshold lookup. The fixture authoring is a task-list item, not a research item.
