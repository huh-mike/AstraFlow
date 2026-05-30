# HTTP API Contract: Chat

**Feature**: 002-chat-interface · **Date**: 2026-05-30

One new streaming endpoint backs the chat surface. It reuses `001`'s module endpoints internally; it does not replace them. The former walkthrough endpoints from `001` remain available.

---

## `POST /api/astra/chat`

Streaming chat turn. Accepts the user message + conversation context; streams back the agent turn (text tokens + inline card parts).

### Request

```jsonc
{
  "userId": "user_001",
  "conversationId": "conv_user_001_001",  // omitted on first turn → server creates one
  "message": "Book a hotel in Tokyo for about S$980"
}
```

### Response (streamed, AI SDK UI message stream)

The stream carries ordered **parts**:

- `text` parts — the agent's natural-language reply (streamed token by token).
- `data-card` parts — one per inline card, each:
  ```jsonc
  {
    "type": "data-card",
    "data": {
      "cardId": "card_dec_001",
      "type": "decision",                 // CardType
      "entityId": "dec_japan_hotel_001",  // 001 entity id
      "displayState": "pending"           // null unless an ask decision card
    }
  }
  ```
  The client `InlineCard` component fetches/binds the referenced `001` entity and renders the existing component.

### Behavioral guarantees

| Guarantee | Source |
|---|---|
| A well-formed request streams a `decision` card within 10s | SC-002 |
| Missing-field request streams a clarifying-question `text` part and **no** `decision` card | FR-005 |
| Smalltalk streams only a `text` part — no card, no order | FR-006 |
| Decision verdict comes from `001`'s `decisionEngine.compose()` only | Principle I |
| `allow`/approved `ask` execution reads only the official quote | Principle III |
| Message appended in arrival order even if sent mid-decision | FR-016 |
| Every decision/execution/feedback emits `001`'s existing events | FR-018, Principle V |

### Errors

| Condition | Response |
|---|---|
| Quote unavailable | `text` part: "quote unavailable"; no `allow`/execution (Edge case, `001` parity) |
| Approve on expired `ask` | `text` part: window closed + offer to re-submit; no execution |
| LLM intent parse failure | retry once → fallback smalltalk clarifying reply; never a fabricated decision |

---

## `GET /api/astra/chat?userId=user_001`

Returns the persisted conversation for reload restoration (FR-015).

### Response

```jsonc
{
  "conversationId": "conv_user_001_001",
  "messages": [
    { "messageId": "msg_user_001_001", "role": "user",  "text": "...", "cards": [], "createdAt": "..." },
    { "messageId": "msg_agent_001_002", "role": "agent", "text": "...", "cards": [ /* EmbeddedCard[] */ ], "createdAt": "..." }
  ]
}
```

Order is chronological by `createdAt` / insertion (FR-002, FR-015).

---

## Reused `001` endpoints (unchanged)

The chat orchestrator calls these server-side; their contracts are defined in `specs/001-consumption-decision-agent/contracts/http-api.md`:

`/api/astra/profile/*`, `/api/astra/goals/*`, `/api/astra/authorizations`, `/api/astra/quotes`, `/api/astra/fit-check`, `/api/astra/decision`, `/api/astra/execute`, `/api/astra/feedback`, `/api/astra/events`.
