<!--
Sync Impact Report
==================
Version change: TEMPLATE (uninitialized) → 1.0.0
Bump rationale: Initial ratification of the AstraFlow constitution. All template
placeholders were replaced with concrete values derived from
`consumption-decision-agent-technical-design.en.md`.

Principles defined:
  I.   Decision Engine Authority (Rule Engine Owns Final Decision) [NON-NEGOTIABLE]
  II.  Authorization Boundary Enforcement [NON-NEGOTIABLE]
  III. Official Quote Is The Only Executable Quote
  IV.  Structured, Schema-First Module Contracts
  V.   Mock-First Demo Scope & Auditable Event Trail

Added sections:
  - Core Principles (I–V)
  - Demo Scope & Technology Constraints
  - Development Workflow & Quality Gates
  - Governance

Removed sections: None (initial creation).

Templates requiring updates:
  - .specify/templates/plan-template.md            ⚠ pending (generic
    "Constitution Check" placeholder still acceptable; refine when first
    feature plan is authored)
  - .specify/templates/spec-template.md            ⚠ pending (review on
    first feature spec)
  - .specify/templates/tasks-template.md           ⚠ pending (review on
    first task generation)
  - .specify/templates/checklist-template.md       ⚠ pending
  - README.md                                       ⚠ pending (no README yet)

Follow-up TODOs: None — RATIFICATION_DATE set to today (2026-05-27) since this is
the first adoption.
-->

# AstraFlow Constitution

## Core Principles

### I. Decision Engine Authority (Rule Engine Owns Final Decision) [NON-NEGOTIABLE]

The Decision Engine is the sole authority that emits the final `allow` / `ask` /
`deny` outcome for a consumption request. The LLM and AI Fit Check module MUST
NOT short-circuit, override, or replace this composition step. LLM output is
treated as advisory context (profile reasoning, soft risks, explanations) and
MUST be funnelled through the Decision Engine, never to the executor directly.

Rationale: A deterministic rule engine is the only component that can guarantee
predictable, auditable, and reproducible authorization behavior. Allowing an LLM
to make the binding decision would compromise user trust and the legal/financial
boundary that authorization policies define.

### II. Authorization Boundary Enforcement [NON-NEGOTIABLE]

Every consumption request MUST be evaluated against an explicit
`AuthorizationPolicy` (standing or temporary) before any execution path runs.
Hard checks — authorization existence, expiry, per-transaction limit, total
limit, category scope, execution frequency — MUST be deterministic, MUST NOT
depend on the LLM, and MUST be evaluated before the AI Fit Check. A failed hard
check MUST yield `deny` without further LLM invocation.

Rationale: Authorization is a contract between the user and the agent. Rule-only
evaluation makes the boundary inspectable and prevents probabilistic models from
silently widening user-granted scope.

### III. Official Quote Is The Only Executable Quote

Order placement and payment MUST be based exclusively on the official quote
returned by the (mock) Merchant MCP. Market intelligence from the consumer
agent network is a reference signal for fit-check and explanation only; it MUST
NOT be passed to the execution path as a transactable price.

Rationale: Conflating market reference data with executable quotes risks
charging the user an amount that no merchant has agreed to honor. Separation
keeps the value flow legally and operationally clean.

### IV. Structured, Schema-First Module Contracts

Every module boundary — Profile Analysis, Financial Goal, Authorization Engine,
Quote Aggregation, AI Fit Check, Decision Engine, Execution Simulation, Feedback
— MUST exchange data using the structured JSON shapes defined in the technical
design (`UserBehaviorProfile`, `FinancialGoalContext`, `AuthorizationPolicy`,
`ConsumptionRequest`, `QuoteResult`, `AIFitCheckResult`, `DecisionResult`,
`ExecutionResult`, `FeedbackRecord`). LLM outputs that feed a downstream module
MUST be coerced into the documented schema before crossing the boundary.

Rationale: Schema-first contracts make the decision flow testable, allow
modules to evolve independently, and let later phases (radar charts, audit
views, replay tooling) consume the same data without re-derivation.

### V. Mock-First Demo Scope & Auditable Event Trail

During the demo / hackathon stage, all external integrations (bank APIs, Apple
Pay history, Merchant MCP, Wallet MCP, consumer agent network, real payment)
MUST be implemented as mocks; no real funds movement is permitted. Every
significant transition (`profile.generated`, `authorization.created`,
`official_quote.received`, `ai_fit_check.completed`, `decision.made`,
`order.created`, `payment.simulated`, `feedback.received`, `context.updated`)
MUST emit an event into the event log so the end-to-end flow is replayable and
auditable.

Rationale: A mock-first stance keeps demo scope honest and risk-free. The event
log is the cheapest substitute for full observability and is what allows
reviewers to verify Principles I–IV were respected on any given run.

## Demo Scope & Technology Constraints

The current product target is the Japan-trip demo scenario (5–7 days, S$2000–
2500 budget) covering the flow: consumption authorization → consumption decision
→ consumption order placement. Implementations MUST keep the scope contained:

- No real bank, card, Apple Pay, Merchant MCP, Wallet MCP, or agent-discovery
  integrations.
- No general product-recommendation engine; the demo's purpose is to make a
  decision, not to rank products.
- Frontend + backend split is permitted (`apps/web`, `apps/api`) but a single
  web app with API routes is equally acceptable as long as module boundaries
  remain explicit.
- Mock data sources (JSON fixtures or in-memory generators) are the canonical
  source of user transactions, financial accounts, goals, authorization
  policies, merchant quotes, market intelligence, and wallet results.

## Development Workflow & Quality Gates

1. Every feature plan (`/speckit-plan` output) MUST include a Constitution Check
   that explicitly demonstrates compliance with Principles I–V or documents a
   justified deviation in the Complexity Tracking table.
2. Any task that touches the Decision Engine, Authorization Engine, or
   Execution Simulation MUST be reviewed for Principle I and II compliance
   before merge.
3. Changes that introduce a new external integration MUST first land as a mock
   behind the existing module boundary; promotion to a real integration is
   out of scope for the current constitution version and requires an
   amendment.
4. Event-log emission for the events listed in Principle V is a merge gate for
   any task that adds or modifies a flow step.

## Governance

This constitution supersedes ad-hoc conventions for AstraFlow. Amendments MUST:

- Be proposed as a PR that updates `.specify/memory/constitution.md` and any
  dependent templates under `.specify/templates/`.
- Include a Sync Impact Report (as an HTML comment at the top of this file) and
  a version bump following semantic versioning:
  - MAJOR — removing or redefining a NON-NEGOTIABLE principle, or changing the
    authority of the Decision Engine / Authorization Engine.
  - MINOR — adding a new principle or section, or materially expanding
    guidance.
  - PATCH — wording, clarification, typo, or non-semantic refinement.
- Be reviewed by at least one project maintainer before merge.

Compliance review is expected at three checkpoints: feature spec finalization,
plan approval, and pre-merge of implementation tasks. Violations discovered in
production demo runs MUST be filed as follow-up issues and resolved before the
next demo iteration.

**Version**: 1.0.0 | **Ratified**: 2026-05-27 | **Last Amended**: 2026-05-27
