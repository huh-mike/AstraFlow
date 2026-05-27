# Feature Specification: User-Profile-Based Consumption Decision Agent

**Feature Branch**: `001-consumption-decision-agent`

**Created**: 2026-05-27

**Status**: Draft

**Input**: User description: "Refer to design doc: consumption-decision-agent-technical-design.en.md — User-profile-based consumption decision agent. End-to-end hackathon demo covering consumption authorization → consumption decision → consumption order placement, walked through the Japan trip scenario (5-7 days, S$2000-2500 budget)."

## Clarifications

### Session 2026-05-27

- Q: What value should the auto-execution threshold (the amount below which `allow` is auto-executed without an `ask` prompt, given hard rules pass and AI fit check is `pass`) take? → A: The threshold is not a fixed constant. The profile engine derives it per user from that user's consumption pattern (typical/median spend by category, historical comfort with automation, budget-discipline trait). The Profile module exposes the derived value as part of `UserBehaviorProfile`; the Decision Engine reads it from the profile for each request.
- Q: When hard rules pass but the AI fit check returns `fail`, what should the final decision be? → A: Severity-scaled by the profile-derived auto-execution threshold: if the request amount is at or above the per-category threshold → `deny`; if the request amount is below the threshold → `ask with strong warning`.
- Q: What happens if a user never responds to an `ask` approval prompt? → A: The pending `ask` expires after a fixed 15-minute window. On expiry, no order is created, no payment is simulated, the system emits a `decision.expired` event to the audit log, and the request is shown as `pending → expired` on next load. The user can re-submit the original request to start a fresh decision.
- Q: Should the v1 user-profile radar-chart dimensions be locked to a canonical set, or determined by the LLM per user? → A: LLM-determined per user. There is no fixed v1 dimension set; the Profile module's LLM step proposes dimension names + values per user, and downstream consumers (UI radar chart, AI fit check, Decision Engine threshold derivation) treat the dimension map as data, not a fixed schema. The contract is structural only: at least 4 named numeric/ordinal dimensions per profile, stable for the life of a profile snapshot.
- Q: What defines the "trip" window used by `AuthorizationPolicy.executionFrequency` for counting executions against `maxCount`? → A: The trip window equals the authorization's own validity window: any execution timestamped on/after the authorization's `createdAt` and on/before `validUntil` (and tied to that authorization) counts toward `maxCount`. No new fields and no separate `Trip` entity are needed for v1.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - End-to-end allow/ask/deny decision for a consumption request (Priority: P1)

A user has set up a Japan-trip travel authorization (temporary, capped) and a consumption agent submits a request to book a hotel within that trip. The system pulls the user's behavior profile, financial-goal context, authorization policy, an official merchant quote, and a market-intelligence reference, then produces a single clear decision — **allow**, **ask**, or **deny** — with a plain-language explanation. If the decision is *allow*, the order is placed and payment is simulated. If *ask*, the user is shown an approval prompt. If *deny*, the user sees the reason and no transaction occurs.

**Why this priority**: This is the product's core promise. Without a working allow/ask/deny decision driven by profile + goal + authorization + quote + market reference, the demo has nothing to show. Every other story exists to feed this one.

**Independent Test**: Submit a mock consumption request for the Japan hotel scenario; verify that the system returns exactly one decision (allow/ask/deny), that the decision references the authorization policy and the AI fit-check result, and that the corresponding next action (execute / prompt user / show denial reason) is taken.

**Acceptance Scenarios**:

1. **Given** a Travel temporary authorization with a S$1,200 per-transaction limit and a passing AI fit check, **When** a consumption agent requests a S$980 hotel booking, **Then** the system returns `allow`, simulates order placement and payment, and surfaces a confirmation with the merchant order ID and simulated wallet payment ID.
2. **Given** the same authorization but an AI fit check marked `caution` (e.g., quote is high vs. market average), **When** the same request is submitted, **Then** the system returns `ask`, presents the user with an approval card containing the reason and the market-reference comparison, and waits for the user's confirmation before any execution.
3. **Given** a request whose amount exceeds the per-transaction authorization cap, **When** the request is submitted, **Then** the system returns `deny`, no order is created, no payment is simulated, and the user sees the specific rule that was violated.
4. **Given** a request that fails the AI fit check (`fail`) and the hard rules otherwise pass, **When** the request amount is at or above the user's profile-derived per-category auto-execution threshold, **Then** the system returns `deny` with a natural-language explanation of the soft risk that triggered the failure; **When** the amount is below that threshold, **Then** the system returns `ask with strong warning` instead.

---

### User Story 2 - View user profile, financial goal, and avatar state (Priority: P2)

The user opens the demo and sees their consumption profile (profile tags, multi-dimensional traits, avatar mood + message) and their financial-goal context (monthly budget, available travel budget, long-term savings goal, and the impact of the Japan trip on that goal). This is the context the decision engine will use, made visible to the user.

**Why this priority**: Without showing the profile and goal context, the decision result looks arbitrary to demo viewers. Showing them first makes the eventual decision explainable. It's a prerequisite for trust but not for the decision itself, so it sits below the decision flow.

**Independent Test**: Load the demo for a mock user (e.g., `user_001`); verify the page shows profile tags, the avatar's mood/message, multi-dimensional profile traits, the monthly/travel budget figures, and a sentence describing how the Japan trip affects the long-term savings goal.

**Acceptance Scenarios**:

1. **Given** mock data exists for `user_001`, **When** the user opens the demo, **Then** the page displays profile tags (e.g., "travel-positive", "budget-aware"), an avatar with a mood and a one-line message, and at least four LLM-determined named profile dimensions (names not fixed across users) rendered in a radar-chart layout.
2. **Given** the user's monthly budget is S$3,200 and available travel budget is S$2,500, **When** the goal section is rendered, **Then** the user sees both figures and a goal-impact sentence stating whether the Japan trip is affordable under the long-term savings goal.

---

### User Story 3 - Create/view a temporary authorization for the trip (Priority: P2)

The user sets up (or reviews) a temporary Travel authorization scoped to the Japan trip: category, per-transaction limit, total trip limit, validity window, max execution count, and execution mode (manual / auto-by-amount). The authorization is the hard rule boundary the decision engine enforces.

**Why this priority**: The authorization is what the hard-rule layer checks against. Without it, the demo cannot show meaningful deny outcomes from the rule engine. P2 because reasonable demo defaults can be pre-seeded, but the user-visible setup is what makes the "authorization boundary" concept concrete.

**Independent Test**: Open the authorization screen, create a Travel authorization with the demo's stated caps (e.g., S$1,200 single / S$2,500 total, valid until 2026-06-30, max 6 executions); verify the saved policy is shown and is the one consulted by the next decision request.

**Acceptance Scenarios**:

1. **Given** the user is on the authorization setup screen, **When** they create a Travel authorization with the stated caps and validity, **Then** the policy is saved and visible, and a subsequent consumption request references that policy ID in its decision result.
2. **Given** an authorization is expired or for the wrong category, **When** a consumption request is evaluated against it, **Then** the hard-rule layer rejects the request before the AI fit check runs and the decision is `deny` with a reason naming the failing rule.

---

### User Story 4 - See official quote alongside market intelligence (Priority: P2)

For each consumption request the user can see two distinct things: (a) the **official quote** that would actually be transacted (from the mock Merchant MCP), and (b) **market intelligence** from the mock consumer-agent network (market average, price range, whether the official quote sits inside the range). The official quote is the only thing used for order placement; market intelligence only informs whether the quote is reasonable.

**Why this priority**: The two-source distinction is part of the product's stated value — agents shop on official quotes but reason with market context. Visualising both makes the decision card believable.

**Independent Test**: Trigger a consumption request and verify the UI shows the official quote (merchant, amount, executable=true) and a separate market-intelligence panel (average, low/high range, signal such as `within_market_range`) — and that the placed order, if any, uses the official-quote amount, not the market average.

**Acceptance Scenarios**:

1. **Given** the mock Merchant MCP returns a S$980 quote and the mock consumer-agent network reports a market average of S$910 (range S$820-1100), **When** the quote panel is rendered, **Then** both numbers are shown side-by-side and a signal indicates the quote is within market range.
2. **Given** the official quote is well above the market high, **When** the AI fit check runs, **Then** the explanation mentions the price gap and the resulting fit check is at least `caution`.

---

### User Story 5 - Feedback loop updates user context after a decision (Priority: P3)

After an execution (or a denial), the user gives short feedback ("this matched my plan" / thumbs up / thumbs down / free text). The system records the feedback, extracts key signals (e.g., "user accepts mid-range hotel for Japan trip", "travel auto-execution confidence increased"), and updates the user context so that future decisions can shift accordingly.

**Why this priority**: Closes the consumption-feedback loop the design doc calls out, but the demo can ship without it and still tell the core story. P3.

**Independent Test**: After an `allow` decision is executed, submit user feedback; verify a feedback record is stored, that one or more context signals are extracted, and that the user context / profile snapshot reflects those signals on next load.

**Acceptance Scenarios**:

1. **Given** an execution has completed, **When** the user submits positive feedback, **Then** a feedback record is stored, at least one context signal is derived, and the updated context is visible on the next profile view.

---

### User Story 6 - Audit trail / event log for the decision flow (Priority: P3)

The user (or demo presenter) can view a chronological event log of the entire flow — profile generated, goal context loaded, authorization created, official quote received, market intelligence received, AI fit check completed, decision made, order created, payment simulated, feedback received, context updated.

**Why this priority**: Strong storytelling aid for the demo and a foundation for future audit features, but not required for the headline decision flow. P3.

**Independent Test**: Run the end-to-end flow once; verify the event log shows the expected events in order with timestamps, and that each event names the artifact it produced (e.g., decision ID, order ID).

**Acceptance Scenarios**:

1. **Given** a complete end-to-end run, **When** the event log is viewed, **Then** all eleven canonical events appear in order, each with a timestamp and a reference to the entity it produced or consumed.

### Edge Cases

- **Expired authorization**: Request arrives after `validUntil` → hard rule denies, no AI call.
- **Category mismatch**: A `Travel` authorization with a `Daily` request → hard rule denies.
- **Frequency exhausted**: 7th request against a `maxCount: 6` authorization, counting only executions whose timestamp lies within the authorization's `createdAt → validUntil` window (per FR-004a), → hard rule denies.
- **Single-amount over limit but total under limit**: Per-transaction cap takes precedence → deny.
- **AI fit check fails but amount is below the profile-derived threshold**: Decision becomes `ask with strong warning` (per FR-012) rather than `deny` or silent `allow`.
- **AI fit check fails and amount is at or above the profile-derived threshold**: Decision becomes `deny` per FR-012.
- **Official quote unavailable / merchant offline**: No `allow` is possible (final order placement requires an official quote). System surfaces a "quote unavailable" state rather than falling back to market intelligence for execution.
- **Market intelligence unavailable**: Decision can still proceed; the AI fit check explanation notes the missing reference.
- **User denies an `ask` prompt**: No order is placed; feedback loop records the denial as a signal.
- **User never responds to an `ask` prompt**: The pending request expires after 15 minutes (per FR-015); no order is placed, a `decision.expired` event is logged, and the user can re-submit the original request to restart the decision.
- **Two consumption requests arrive concurrently against the same authorization**: System evaluates them against the authorization state at the time of each request (demo-acceptable: serial evaluation).

## Requirements *(mandatory)*

### Functional Requirements

**Profile & Context**

- **FR-001**: System MUST generate a structured user behavior profile (profile tags, avatar state with mood + message, multi-dimensional traits, **and a per-category auto-execution threshold derived from the user's consumption pattern**) from the user's mock consumption, income, expense, budget, goal, and feedback data.
- **FR-001a**: System MUST derive the auto-execution threshold per user (and per spending category) from the user's consumption pattern — e.g., typical/median spend in that category, historical comfort with automation, and the user's budget-discipline signals — and MUST expose it as a field on `UserBehaviorProfile` consumable by the Decision Engine.
- **FR-001b**: The profile's multi-dimensional traits MUST be **LLM-determined per user** (no fixed v1 dimension set). The Profile module MUST output at least 4 named numeric/ordinal dimensions per profile; downstream consumers (UI radar chart, AI fit check, Decision Engine threshold derivation) MUST treat the dimension map as data rather than a fixed schema. The dimension names and value scale for a given profile snapshot MUST remain stable for the lifetime of that snapshot.
- **FR-002**: System MUST present a financial-goal context including monthly budget, available trip budget, at least one long-term savings goal, and a sentence describing the impact of the demo scenario on that goal.

**Authorization**

- **FR-003**: Users MUST be able to create a temporary authorization scoped to a category (e.g., Travel) with: per-transaction limit, total limit, currency, validity window (`createdAt` → `validUntil`), max execution count within that window, and an execution mode (manual / auto-by-amount).
- **FR-004**: System MUST enforce all authorization dimensions (amount limits, category scope, frequency, validity window) deterministically via a rule engine, without LLM involvement.
- **FR-004a**: For `executionFrequency.maxCount` enforcement, the rule engine MUST count every execution tied to a given authorization whose timestamp falls on/after that authorization's `createdAt` and on/before its `validUntil`. The label `window: "trip"` in `AuthorizationPolicy` is shorthand for this validity-window-bounded count; no separate trip start/end fields or `Trip` entity are required in v1.
- **FR-005**: System MUST distinguish standing authorizations (always-on, broader caps) from temporary authorizations (trip-scoped).

**Quotes**

- **FR-006**: System MUST retrieve an official executable quote (merchant name, amount, currency, executable flag, quote ID) for each consumption request from the mock Merchant MCP.
- **FR-007**: System MUST retrieve a market-intelligence reference (market average, low/high range, range-fit signal) from the mock consumer-agent network for each consumption request.
- **FR-008**: System MUST use only the official quote — never the market-intelligence average — when placing an order.

**AI Fit Check**

- **FR-009**: System MUST run an AI fit check that outputs exactly one of three levels (`pass`, `caution`, `fail`), an optional list of soft-risk notes, and a natural-language explanation.
- **FR-010**: The AI fit check MUST consider profile match, financial-goal alignment, quote reasonableness vs. market reference, and soft risks. It MUST NOT make the final allow/ask/deny decision.

**Decision**

- **FR-011**: System MUST compose a final decision of exactly one of `allow`, `ask`, `deny`, with: a reason string, a `requiresUserApproval` flag, and a `nextAction` (e.g., `execute_order`, `request_user_approval`, `show_denial`).
- **FR-012**: Decision composition MUST follow the deterministic rule:
  - hard-rule fail → `deny` (no AI fit check needed)
  - hard-rule pass + AI `pass` + amount below the **profile-derived per-category auto-execution threshold** (see FR-001a) → `allow`
  - hard-rule pass + AI `pass` + amount at or above that threshold → `ask`
  - hard-rule pass + AI `caution` → `ask`
  - hard-rule pass + AI `fail` + amount **at or above** the profile-derived per-category auto-execution threshold → `deny`
  - hard-rule pass + AI `fail` + amount **below** that threshold → `ask with strong warning`
- **FR-013**: The final allow/ask/deny decision MUST be owned by the rule engine; the LLM contributes inputs and explanations only.

**Execution**

- **FR-014**: When the decision is `allow`, System MUST simulate order creation via the mock Merchant MCP and payment via the mock Wallet MCP, producing an execution record with an order ID, payment ID, order status, and payment status.
- **FR-015**: When the decision is `ask`, System MUST present an approval prompt to the user containing the reason and the supporting context (official quote + market reference + fit-check explanation), and MUST NOT execute until the user approves. The pending `ask` MUST expire 15 minutes after issuance if the user has not approved or declined; on expiry the system MUST NOT create an order, MUST emit a `decision.expired` event, and MUST surface the request as `pending → expired` on next load.
- **FR-016**: When the decision is `deny`, System MUST surface the failing rule(s) and/or fit-check reason to the user and MUST NOT create an order or simulate payment.

**Feedback**

- **FR-017**: After any decision outcome, users MUST be able to submit feedback (free text and/or a short reaction), which the system stores as a feedback record linked to the execution or decision.
- **FR-018**: System MUST derive one or more context signals from each feedback submission and apply them to update the user context / profile snapshot.

**Audit / Events**

- **FR-019**: System MUST emit a chronological event log covering: `profile.generated`, `goal.context.loaded`, `authorization.created`, `official_quote.received`, `market_intelligence.received`, `ai_fit_check.completed`, `decision.made`, `decision.expired` (when an `ask` is not actioned within its 15-minute window), `order.created`, `payment.simulated`, `feedback.received`, `context.updated`.

**Demo UI flow**

- **FR-020**: The demo UI MUST present the flow in the following order on a single walkthrough: profile + avatar → goal & budget → authorization setup → official quote + market intelligence → decision card → execution trace (or approval / denial state) → feedback + context update.

**Scope guardrails**

- **FR-021**: All inputs (user data, financial accounts, merchant quotes, market intelligence, wallet payments) MUST be served from mock data sources for this demo; no real bank, card, merchant, wallet, or agent-network integrations are in scope.

### Key Entities *(include if feature involves data)*

- **UserBehaviorProfile**: Structured snapshot of a user's consumption behavior — profile tags, avatar state (mood + message), an LLM-determined multi-dimensional trait map (≥4 named numeric/ordinal dimensions per user; names not fixed across users; stable for the snapshot's lifetime), and a per-category auto-execution threshold map derived from the user's consumption pattern (consumed by the Decision Engine).
- **FinancialGoalContext**: Monthly budget, scenario-scoped available budget, one or more long-term savings goals (with target + current amounts), and a goal-impact narrative for the current scenario.
- **AuthorizationPolicy**: A user-issued boundary identified by ID, with type (standing/temporary), category, single/total caps, currency, validity window (`createdAt` → `validUntil`), execution frequency (max count, with the count window equal to the validity window per FR-004a), and execution mode.
- **ConsumptionRequest**: A request submitted by an agent on the user's behalf — request ID, user ID, agent ID, category, purpose, amount, currency.
- **QuoteResult**: A pair of (a) official quote (merchant, amount, executable flag) and (b) market intelligence (average, range, signal).
- **AIFitCheckResult**: The AI's contribution — a three-level fit check (`pass`/`caution`/`fail`), optional soft-risk notes, and a plain-language explanation.
- **DecisionResult**: The final verdict — one of `allow`/`ask`/`deny`, with reason, approval-required flag, and next action.
- **ExecutionResult**: The simulated outcome — execution ID, order status, payment status, merchant order ID, wallet payment ID.
- **FeedbackRecord**: A user's post-decision response, linked to the execution, with extracted context signals.
- **Event**: A timestamped record of a step in the decision flow, used for the audit trail.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: A presenter can walk through the full Japan-trip demo (profile → goal → authorization → quote → decision → execution → feedback) in under 5 minutes without manual data entry beyond the authorization step.
- **SC-002**: For each consumption request, the system produces exactly one decision (`allow`, `ask`, or `deny`) and the corresponding next action runs end-to-end in under 10 seconds.
- **SC-003**: All eleven canonical events appear in the audit log for a complete `allow` run, in order, with no missing or duplicate events.
- **SC-004**: 100% of `allow` outcomes execute against the official quote amount (never the market average), verifiable from the execution record.
- **SC-005**: 100% of authorization-rule violations (expired, wrong category, over single-amount cap, over total cap, frequency exhausted) result in `deny` without invoking the AI fit check.
- **SC-006**: For each `ask` and `deny` outcome, the user sees a plain-language explanation referencing the specific rule and/or soft risk that caused it (verified by inspection on at least three distinct scenarios).
- **SC-007**: After submitting feedback, the updated user context is visible on the next profile view within one page reload and contains at least one signal derived from the feedback.

## Assumptions

- The demo runs against mock data only — no real bank APIs, no real Apple Pay / card history, no real Merchant MCP, no real Wallet MCP, no real consumer-agent network, no real payment.
- A single demo user (`user_001`) and a single demo scenario (Japan trip, 5-7 days, S$2,000-2,500 budget) are sufficient for acceptance; multi-user, multi-scenario, and multi-currency support are out of scope for v1.
- The "auto-execution threshold" is **not** a configured constant — the profile engine derives it per user (and per category) from the user's consumption pattern (typical/median spend, historical comfort with automation, budget-discipline trait) and surfaces it through `UserBehaviorProfile`. The Decision Engine reads it from the profile per request rather than from global config.
- The UI runs as a web demo; mobile-native is out of scope for v1.
- Product-recommendation features and complex financial-planning productization are explicitly out of scope (see Non-Goals in the source design doc).
- The system runs the AI fit check and the rule engine sequentially per request; concurrent same-authorization requests are handled by serial evaluation in the demo.
- Authentication is out of scope for v1 — the demo loads a pre-seeded mock user.
- Profile dimensions are LLM-determined per user (no fixed v1 schema); see FR-001b. The radar-chart UI must render whichever dimensions the LLM produced for the loaded profile snapshot.
