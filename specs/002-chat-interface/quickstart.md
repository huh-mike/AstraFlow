# Quickstart: Chat Interface

**Feature**: 002-chat-interface · **Date**: 2026-05-30

This feature layers a chat surface on top of `001-consumption-decision-agent`. Prerequisites and environment are inherited from `001` (Next.js 16, Node 22, `AI_GATEWAY_API_KEY`).

## Prerequisites

- `001` modules present and green (decision pipeline, mock MCPs, schemas, event log).
- `.env.local` with `AI_GATEWAY_API_KEY` (same as `001`). Set `DEMO_LIVE_LLM=true` for live intent classification; unset uses deterministic intent fixtures.

## Run

```bash
npm install
npm run dev            # http://localhost:3000  → chat thread is the primary surface
# secondary/fallback walkthrough still available at /walkthrough
```

## Demo script (single conversation — SC-008, under 5 min)

1. **Open `/`** — greeted by the chat thread for `user_001`.
2. **Type** "Show me my profile and budget" → agent posts a **profile card** (tags, avatar, ≥4 radar dimensions) and a **goal/budget card** (US3, FR-003).
3. **Type** "Set up a Travel authorization for the Japan trip, S$1,200 single / S$2,500 total" → agent posts an **authorization card** (US3).
4. **Type** "Book a hotel in Tokyo for about S$980" → agent interprets the request and posts a **decision card** with explanation, plus a **quote card** (official vs. market) (US1, FR-004, FR-007).
5. **If `ask`** → approve inline ("yes, go ahead" or the card button) → agent posts an **execution card** (order id + simulated payment) (US2, FR-009, FR-010).
6. **Type** "Why was that the price — is it reasonable?" → grounded follow-up answer citing the market reference, no new verdict (US4, FR-012).
7. **React** 👍 or type "that matched my plan" → agent acknowledges, stores feedback, confirms context update (US5, FR-014).
8. **Scroll up / reload** → full ordered thread with cards is restored (US6, FR-015).

## Verify the constitution boundaries

- **Decision authority (I)**: trigger a hard-rule violation in chat (e.g., "book a S$5,000 hotel" over the cap) → `deny` card with the failing rule; no fit check ran.
- **Approval authority (I/FR-011)**: reply "maybe" to an `ask` → agent re-prompts; no execution.
- **Official quote only (III)**: inspect the execution card → amount equals the official quote, never the market average.
- **Events (V)**: open `/api/astra/events` → the same canonical `001` events appear for the chat-driven run; no new event types.

## Test

```bash
npm run test:unit       # includes intent-extraction, approval-mapping, conversation-ordering
npm run test:contract   # includes chat-intent contract; 001 contracts remain green (unchanged)
npm run test:e2e        # chat-happy-path.spec.ts: NL request → decision → approve → execution → feedback
```

## What changed vs. 001

- New primary surface: `app/page.tsx` (chat). Former walkthrough moved to `app/walkthrough/`.
- New: `app/api/astra/chat/route.ts`, `lib/modules/chat/*`, `lib/schemas/{message-intent,conversation}.ts`, chat components under `app/components/chat/`.
- Reused unchanged: all `001` decision-critical modules, schemas, mocks, and event log.
