# Feature Specification: Chat Interface as the Main User Experience

**Feature Branch**: `002-chat-interface`

**Created**: 2026-05-30

**Status**: Draft

**Input**: User description: "Add chat interface as the main UX for the user."

## Context

This feature changes the primary surface of the consumption decision agent (feature `001-consumption-decision-agent`) from a fixed, card-based single-page walkthrough into a **conversational chat interface**. The user now interacts with the agent the way they would with a messaging app: they type natural-language messages, the agent replies in the same thread, and the structured artifacts the product already produces — profile, financial goal, authorization, official quote + market intelligence, decision card, execution trace, feedback — are surfaced as **rich inline cards embedded inside the conversation** rather than as separate fixed page sections.

The underlying decision logic (hard-rule engine, AI fit check, allow/ask/deny composition, mock MCP execution) defined in feature `001` is unchanged and is the dependency this feature consumes. This feature governs only the **conversational presentation and interaction layer** on top of it.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Initiate a consumption request through natural-language chat (Priority: P1)

The user opens the demo and is greeted by a chat thread. They type a free-form message describing what they want — e.g., "Book me a hotel in Tokyo for the trip, around S$1,000." The agent interprets the message into a structured consumption request, runs the existing decision flow, and replies in the thread with a decision card (allow/ask/deny) plus a plain-language explanation, all without the user filling in any structured form.

**Why this priority**: This is the headline of the feature — the chat is the *main* UX. If the user cannot drive a consumption decision by typing a message and seeing the result inline, the chat interface delivers no value. Every other story refines or extends this core loop.

**Independent Test**: Type a natural-language hotel-booking request into the chat for the Japan scenario; verify the agent posts back, in the same thread, a single decision card (allow/ask/deny) with an explanation, and that the decision matches what the underlying decision engine would return for the equivalent structured request.

**Acceptance Scenarios**:

1. **Given** an empty chat thread for `user_001`, **When** the user types "Book a hotel in Tokyo for about S$980," **Then** the agent posts an assistant message containing a decision card whose verdict (allow/ask/deny) and explanation correspond to the decision engine's output for that request.
2. **Given** the user's message omits a required detail (e.g., no amount), **When** the message is sent, **Then** the agent asks a clarifying question in the thread and does not produce a decision until the missing detail is supplied.
3. **Given** the user types a message unrelated to a consumption request (e.g., "hello"), **When** the message is sent, **Then** the agent responds conversationally and does not fabricate a decision or place an order.

---

### User Story 2 - Approve, decline, or let an `ask` decision expire — inline in chat (Priority: P1)

When the decision is `ask`, the agent posts an approval card in the thread with the reason and supporting context (official quote, market reference, fit-check explanation). The user approves or declines directly in the conversation (a button on the card or a typed reply such as "yes, go ahead"). The thread then shows the resulting execution or the recorded decline. If the user never responds, the card visibly transitions to an expired state per the underlying 15-minute rule.

**Why this priority**: The `ask` path is half of the product's core promise and the most interactive moment. In a chat UX the approval must happen *in the conversation*, not on a separate screen, or the experience fragments. Tied with US1 as the critical loop.

**Independent Test**: Drive a request that yields `ask`; verify the approval card appears inline, that approving it posts an execution result in the thread, that declining it posts a recorded decline, and that leaving it unanswered shows an expired state after the decision window.

**Acceptance Scenarios**:

1. **Given** an `ask` approval card is shown in the thread, **When** the user approves it, **Then** the agent posts an execution result message (order ID + simulated payment ID) below it and the card updates to an "approved" state.
2. **Given** an `ask` approval card is shown, **When** the user declines (via control or typed refusal), **Then** no order is created, the card updates to a "declined" state, and the decline is recorded as feedback.
3. **Given** an `ask` approval card is shown, **When** the user sends no response within the decision window, **Then** the card updates to an "expired" state and the thread reflects that no order was placed.

---

### User Story 3 - Inspect profile, goal, authorization, and quotes as inline cards (Priority: P2)

At relevant points in the conversation — on first load, when the user asks "what's my profile?", or when a decision references them — the agent surfaces the existing structured artifacts as rich cards inside the thread: the profile + avatar + radar dimensions, the financial-goal/budget summary, the active authorization, and the official-quote-vs-market-intelligence panel. The user can read the same information the decision engine used, in context, without leaving the chat.

**Why this priority**: These cards make decisions explainable, but the chat can demonstrate the core loop (US1/US2) before every artifact is richly rendered. P2 because they elevate trust and storytelling rather than gate the headline interaction.

**Independent Test**: Ask "show me my profile and budget" in the chat; verify the agent posts a profile card (tags, avatar mood + message, ≥4 named radar dimensions) and a goal/budget card (monthly budget, trip budget, goal-impact sentence) inline, matching the underlying profile and goal context for `user_001`.

**Acceptance Scenarios**:

1. **Given** the chat is loaded for `user_001`, **When** the user asks to see their profile, **Then** the agent posts a profile card with profile tags, avatar mood + one-line message, and at least four named profile dimensions rendered as a radar visualization.
2. **Given** a decision references an official quote and market reference, **When** the decision card is posted, **Then** the user can view a quote card showing the official quote and the separate market-intelligence panel (average, range, range-fit signal) tied to that decision.
3. **Given** the user asks about their spending authorization, **When** the message is sent, **Then** the agent posts an authorization card showing category, caps, validity window, max executions, and execution mode.

---

### User Story 4 - Ask follow-up questions about a decision (Priority: P2)

After a decision is posted, the user can ask the agent to explain itself in natural language — "why did you deny that?", "what if it were S$800 instead?", "is that price reasonable?". The agent answers using the same decision context (rule outcomes, fit-check notes, market reference) already produced for that request, keeping the explanation grounded in the actual decision rather than inventing new ones.

**Why this priority**: Conversational follow-up is what distinguishes a chat from a static card list and is a major reason to adopt this UX. It is P2 because the core decision loop (US1/US2) is usable without it, but it strongly increases the perceived intelligence and trust of the agent.

**Independent Test**: After a `deny` decision, ask "why was this denied?"; verify the agent's reply references the specific failing rule and/or soft risk from that decision's context and introduces no new verdict.

**Acceptance Scenarios**:

1. **Given** a `deny` decision has been posted, **When** the user asks why, **Then** the agent's reply names the specific authorization rule and/or fit-check reason that drove the denial, consistent with that decision's recorded explanation.
2. **Given** an `allow` decision has been posted, **When** the user asks "was that a good price?", **Then** the agent answers using the market-intelligence reference attached to that request (e.g., within/above market range) without changing the decision.
3. **Given** the user asks a hypothetical follow-up that would require a different amount or authorization, **When** the message is sent, **Then** the agent either clearly frames the answer as hypothetical or offers to submit it as a new request, and does not silently mutate the prior decision.

---

### User Story 5 - Give feedback conversationally after an outcome (Priority: P3)

After any outcome, the user can react or comment in the thread — a thumbs up/down control on the outcome message, or a typed message like "that matched my plan." The agent acknowledges, stores the feedback, derives context signals, and confirms in the conversation that the profile/context was updated.

**Why this priority**: Closes the feedback loop inside the chat, but the demo can ship the conversational decision flow without it. P3, matching the priority of feedback in feature `001`.

**Independent Test**: After an executed `allow`, send positive feedback in the chat; verify a feedback record is stored, at least one context signal is derived, and the agent confirms the context update in the thread.

**Acceptance Scenarios**:

1. **Given** an execution result message is shown, **When** the user reacts positively or types approving feedback, **Then** the agent acknowledges in the thread, stores a feedback record linked to the execution, and surfaces that the user context was updated.

---

### User Story 6 - Persisted, scrollable conversation history (Priority: P3)

The conversation is an ordered, scrollable history of user and agent turns, including the embedded cards, so the user (or a demo presenter) can scroll back through the whole session: requests, decisions, approvals, explanations, and feedback, in the order they happened. Reloading the demo restores the visible thread for the demo user.

**Why this priority**: A coherent, reviewable thread is what makes the chat feel like a real product and doubles as the human-readable narrative of the session. P3 because a single live exchange already demonstrates the core value; persistence and scrollback are enhancements.

**Independent Test**: Run several exchanges, scroll to the top, and reload; verify the full ordered history (messages + cards) is preserved and rendered in chronological order for `user_001`.

**Acceptance Scenarios**:

1. **Given** multiple exchanges have occurred, **When** the user scrolls up, **Then** all prior user messages, agent replies, and embedded cards are visible in chronological order.
2. **Given** an active conversation, **When** the demo is reloaded, **Then** the previously visible thread for the demo user is restored in the same order.

### Edge Cases

- **Ambiguous or underspecified request**: The agent asks a clarifying question rather than guessing critical fields (amount, category); no decision or order is produced until resolved.
- **Non-actionable / off-topic message**: The agent replies conversationally and never fabricates a decision, quote, or order.
- **User edits or resends a similar request**: A new request is created and evaluated against current authorization state; the prior decision card is left intact in the history.
- **User approves an already-expired `ask` card**: The approval is rejected with an explanation that the window closed, and the agent offers to re-submit the request.
- **User sends a new message while a decision is still being computed**: The new message is queued or acknowledged so messages are not lost or interleaved out of order; the in-flight decision still completes and posts.
- **Rapid multiple requests in one message** (e.g., "book a hotel and a flight"): The agent either processes them as separate sequential requests with separate cards or asks the user to take them one at a time; it does not merge them into a single ambiguous order.
- **Quote unavailable mid-conversation**: The agent posts a "quote unavailable" message inline; no `allow`/execution occurs (consistent with feature `001`).
- **Free-text approval ambiguity** (e.g., "maybe"): The agent treats it as not-yet-approved and asks for an explicit confirm/decline rather than executing.

## Requirements *(mandatory)*

### Functional Requirements

**Conversation surface**

- **FR-001**: The chat interface MUST be the primary user-facing surface of the demo; the user MUST be able to reach every core capability of feature `001` (view profile/goal/authorization, submit a consumption request, receive a decision, approve/decline an `ask`, see execution results, give feedback) through the conversation without depending on the fixed card-based walkthrough page.
- **FR-002**: The system MUST present an ordered conversation thread of user and agent turns, rendered newest-in-context with the ability to scroll back through the full session history.
- **FR-003**: The system MUST render the existing structured artifacts (profile + avatar + radar dimensions, financial-goal/budget summary, authorization, official-quote + market-intelligence panel, decision card, execution trace, feedback acknowledgement) as rich cards embedded inline within agent messages, not on a separate page.

**Natural-language requests**

- **FR-004**: The system MUST accept free-form natural-language input from the user and interpret messages that express a consumption intent into a structured consumption request (category, purpose, amount, currency) suitable for the existing decision engine.
- **FR-005**: When a consumption intent is detected but a required field for a decision is missing or ambiguous (e.g., amount or category), the system MUST ask a clarifying question in the thread and MUST NOT produce a decision or place an order until the field is resolved.
- **FR-006**: When a message expresses no consumption intent, the system MUST respond conversationally and MUST NOT fabricate a decision, quote, order, or payment.

**Decision presentation**

- **FR-007**: For each interpreted consumption request, the system MUST run the existing decision flow (hard-rule engine → AI fit check → allow/ask/deny composition) defined in feature `001` and post the resulting decision as a card in the thread with the plain-language explanation. This feature MUST NOT alter the decision logic or its allow/ask/deny rules.
- **FR-008**: The decision card MUST surface the verdict, the reason, and access to the supporting context (official quote, market-intelligence reference, fit-check explanation) for that request.

**Inline approval flow**

- **FR-009**: When the decision is `ask`, the system MUST present an approval card inline in the thread that allows the user to approve or decline both via an on-card control and via a typed natural-language reply, and MUST NOT execute until the user explicitly approves.
- **FR-010**: On inline approval of an `ask`, the system MUST trigger the existing execution simulation and post the execution result (order ID, payment ID, statuses) in the thread; on decline, it MUST record the decline as feedback and post a declined state; on no response within the decision window, it MUST show an expired state — consistent with feature `001`'s 15-minute expiry rule.
- **FR-011**: The system MUST treat ambiguous replies to an approval card (e.g., "maybe") as not-yet-approved and request an explicit confirm or decline.

**Conversational explanation**

- **FR-012**: After a decision is posted, the system MUST answer the user's natural-language follow-up questions about that decision using only that decision's recorded context (rule outcomes, fit-check notes, market reference, profile/goal context) and MUST NOT introduce a new or contradictory verdict for the same request.
- **FR-013**: When a follow-up implies a different amount, category, or authorization, the system MUST frame the response as hypothetical or offer to submit it as a new request, and MUST NOT silently mutate the prior decision or its record.

**Feedback in-thread**

- **FR-014**: The system MUST allow the user to submit feedback on an outcome from within the conversation (reaction control and/or free-text message), store it as a feedback record linked to the execution or decision, derive at least one context signal, and acknowledge the update in the thread — reusing feature `001`'s feedback behavior.

**History & continuity**

- **FR-015**: The system MUST preserve conversation turns (user messages, agent messages, and embedded cards) in chronological order for the demo user, and MUST restore the visible thread on reload.
- **FR-016**: When the user sends a new message while a decision is still being computed, the system MUST preserve ordering so no message is lost or interleaved incorrectly, and the in-flight decision MUST still complete and post its result.

**Continuity with feature `001`**

- **FR-017**: All data shown in the chat (profile, goal, authorization, quotes, market intelligence, execution, payment) MUST come from the same mock data sources and decision engine as feature `001`; no real integrations are introduced by this feature.
- **FR-018**: Every consumption decision, execution, expiry, and feedback event triggered through the chat MUST emit the same audit/event-log entries defined in feature `001` (e.g., `decision.made`, `order.created`, `payment.simulated`, `decision.expired`, `feedback.received`, `context.updated`).

### Key Entities *(include if feature involves data)*

- **Conversation**: An ordered session of interaction for a user — identified by ID, owning user, created timestamp, and an ordered list of messages. Holds the full demo narrative.
- **Message**: A single turn in a conversation — role (user or agent), text content, timestamp, ordering position, and zero or more embedded cards. Links to the request/decision/execution it concerns when applicable.
- **EmbeddedCard**: A structured artifact rendered inside an agent message — a typed reference (profile, goal, authorization, quote, decision, execution, feedback-ack) to an entity defined in feature `001`, plus the display state needed to render it inline (e.g., an approval card's pending/approved/declined/expired state).
- **MessageIntent**: The interpreted meaning of a user message — intent type (consumption request, approval/decline, follow-up question, feedback, smalltalk/other), and, for consumption requests, the extracted structured fields (category, purpose, amount, currency) and any missing-field flags.

*Note: `UserBehaviorProfile`, `FinancialGoalContext`, `AuthorizationPolicy`, `ConsumptionRequest`, `QuoteResult`, `AIFitCheckResult`, `DecisionResult`, `ExecutionResult`, `FeedbackRecord`, and `Event` are reused unchanged from feature `001`.*

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: A user can complete the full core loop — type a natural-language request, receive a decision, and (for `ask`) approve it to execution — entirely within the chat thread, without using any separate fixed-walkthrough page.
- **SC-002**: For a well-formed natural-language consumption request, the agent posts a decision card within 10 seconds, and the verdict matches what the underlying decision engine returns for the equivalent structured request in 100% of test scenarios.
- **SC-003**: For requests missing a required field, the agent asks a clarifying question instead of producing a decision in 100% of such cases (verified across at least three under-specified inputs).
- **SC-004**: 100% of `ask` outcomes can be approved or declined inline in the thread, and approval produces an execution result message referencing the official-quote order — never the market average.
- **SC-005**: For at least three distinct decisions (one allow, one ask, one deny), a natural-language "why?" follow-up returns an explanation that references the specific rule and/or soft risk recorded for that decision, with no new or contradictory verdict.
- **SC-006**: After feedback submitted in-thread, the agent acknowledges and the updated user context is reflected on the next profile view within one reload, containing at least one derived signal.
- **SC-007**: The full conversation history (messages and embedded cards) renders in correct chronological order and is restored after a reload in 100% of test runs.
- **SC-008**: A presenter can narrate the entire Japan-trip demo as a single chat conversation (request → decision → approval/denial → explanation → feedback) in under 5 minutes.

## Assumptions

- This feature builds on and depends on feature `001-consumption-decision-agent`; its decision engine, mock MCPs, profile/goal/authorization data, and event log are reused unchanged. The chat is a new presentation/interaction layer, not a re-implementation of decision logic.
- The chat is the **main** UX, but the existing card artifacts are retained as inline components; the fixed single-page walkthrough from feature `001` may remain available as a secondary/fallback view but is no longer the primary surface.
- A single demo user (`user_001`) and the single Japan-trip scenario remain sufficient for acceptance; multi-user and multi-conversation management are out of scope for v1.
- Natural-language understanding is scoped to the demo's consumption domain (book/buy within the trip): the agent must reliably extract category, purpose, and amount for in-scope requests, but broad open-domain conversation is not a goal.
- Voice input, attachments/images, and multi-modal chat are out of scope for v1; input is typed text.
- Authentication remains out of scope for v1 — the demo loads the pre-seeded mock user, and the conversation belongs to that user.
- The UI runs as a web demo; mobile-native is out of scope for v1.
- Conversation persistence is scoped to restoring the demo user's visible thread for continuity; long-term archival, search, and export of conversations are out of scope for v1.
- Streaming/typing indicators and similar presentation niceties are desirable but not required for acceptance; correctness of content and ordering is what the success criteria measure.
