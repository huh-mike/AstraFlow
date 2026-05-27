# HTTP API Contracts

**Date**: 2026-05-27 · **Feature**: 001-consumption-decision-agent

All routes live under `app/api/` in the Next.js App Router. JSON in / JSON out. All payloads are Zod-validated against the schemas in `data-model.md`. Errors return `{ error: { code, message } }` with appropriate HTTP status.

---

## End-to-end aggregator

### `POST /api/demo/japan-trip/decision-flow`

The "single end-to-end API" from the design doc §6.1 — the presenter's primary endpoint.

**Request body**:
```json
{
  "userId": "user_001",
  "scenario": "japan_trip",
  "request": "Plan and book a Japan trip for 5-7 days within S$2000-2500."
}
```

**Response 200**:
```json
{
  "profile": { /* UserBehaviorProfile */ },
  "goalContext": { /* FinancialGoalContext */ },
  "authorization": { /* AuthorizationPolicy used */ },
  "quotes": { /* QuoteResult */ },
  "fitCheck": { /* AIFitCheckResult */ },
  "decision": { /* DecisionResult */ },
  "execution": null | { /* ExecutionResult */ },
  "feedbackPrompt": { "promptText": "How did this go?" },
  "events": [ /* Event[] for this correlation */ ]
}
```

**Errors**:
- `404` if `userId` has no fixture.
- `409` if no authorization exists for the resolved category (UI should redirect to authorization setup).

---

## Module endpoints (debugging / direct use)

### `GET /api/astra/profile/:userId`

Returns the latest `UserBehaviorProfile` snapshot, generating one if none exists.

- **200**: `UserBehaviorProfile`
- **404**: unknown user

### `POST /api/astra/profile/analyze`

Force-regenerate a profile snapshot.

- **Body**: `{ "userId": "user_001" }`
- **200**: `UserBehaviorProfile`

### `GET /api/astra/goals/:userId`

- **200**: `FinancialGoalContext`
- **404**: unknown user

### `POST /api/astra/authorizations`

Create a new authorization.

- **Body**: `Omit<AuthorizationPolicy, "authorizationId" | "createdAt">`
- **201**: `AuthorizationPolicy`
- **400**: validation error (e.g., `validUntil ≤ now`)

### `GET /api/astra/authorizations?userId=user_001`

- **200**: `AuthorizationPolicy[]`

### `POST /api/astra/quotes`

- **Body**: `ConsumptionRequest`
- **200**: `QuoteResult`

### `POST /api/astra/fit-check`

- **Body**: `{ request, profile, goalContext, quotes }`
- **200**: `AIFitCheckResult`

### `POST /api/astra/decision`

Run the full decision pipeline for a single request.

- **Body**: `ConsumptionRequest`
- **200**: `{ decision: DecisionResult, execution?: ExecutionResult }`
- **409**: no matching authorization

### `POST /api/astra/decision/:decisionId/approve`

Approve a pending `ask` decision.

- **200**: `{ decision: DecisionResult, execution: ExecutionResult }`
- **404**: unknown decision
- **410**: decision already expired/declined

### `POST /api/astra/decision/:decisionId/decline`

Decline a pending `ask` decision.

- **200**: `{ decision: DecisionResult }`
- **404**: unknown decision
- **410**: decision already expired/approved

### `POST /api/astra/execute`

Manual execution trigger (rarely used; `compose()` calls `execution.simulate()` internally).

- **Body**: `{ decisionId: string }`
- **200**: `ExecutionResult`
- **412**: decision is not in `allow` / `approved` state

### `POST /api/astra/feedback`

- **Body**: `{ decisionId, executionId?, reaction, userFeedback }`
- **201**: `FeedbackRecord`

### `GET /api/astra/events?correlationId=<id>`

Read-only audit log filter.

- **200**: `Event[]`

---

## Authentication

None in v1 (Assumptions). All routes resolve the user from the request body / path param.
