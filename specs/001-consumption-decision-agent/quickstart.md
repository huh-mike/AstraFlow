# Quickstart: Consumption Decision Agent Demo

**Feature**: 001-consumption-decision-agent · **Date**: 2026-05-27

## Prerequisites

- Node.js 22 LTS (`node --version` → `v22.*`)
- pnpm 9+ (`corepack enable && corepack prepare pnpm@latest --activate`)
- A Vercel AI Gateway API key (`AI_GATEWAY_API_KEY`) — needed only for live LLM runs

## 1. Install

```bash
pnpm install
```

## 2. Environment

```bash
cp .env.example .env.local
# Edit .env.local:
# AI_GATEWAY_API_KEY=...        (required for live LLM)
# DEMO_LIVE_LLM=true             (set false to use deterministic fixture responses)
```

## 3. Run the demo

```bash
pnpm dev
# Opens http://localhost:3000
```

Walk-through (matches FR-020):

1. **Profile + Avatar** — load page, see `user_001`'s profile auto-generated.
2. **Goal & Budget** — Japan trip impact sentence.
3. **Authorization Setup** — create a Travel temporary authorization (defaults pre-filled: S$1,200 single / S$2,500 total, valid 2026-06-30, max 6 executions).
4. **Quote & Market** — submit a hotel request; see official quote vs. market reference.
5. **Decision Card** — `allow` / `ask` / `deny`.
6. **Execution / Approval / Denial** trace.
7. **Feedback** — submit; see context update.
8. **Event Log** — view the audit trail.

## 4. Run tests

```bash
pnpm test            # Vitest: unit + contract
pnpm test:e2e        # Playwright: japan-trip happy path
```

## 5. Acceptance checklist

Map to `spec.md` Success Criteria:

- [ ] **SC-001**: Walk-through completes in under 5 minutes.
- [ ] **SC-002**: Each decision returns in under 10 seconds.
- [ ] **SC-003**: A full `allow` run shows all 11 canonical events in order.
- [ ] **SC-004**: Inspect `ExecutionResult.amount` — equals `officialQuote.amount`, not market average. (Asserted by contract test.)
- [ ] **SC-005**: Trigger an expired-authorization request — see `deny` and no `ai_fit_check.completed` event in the log. (Asserted by unit test.)
- [ ] **SC-006**: Three scenarios (`ask`, `deny` by hard rule, `deny` by AI fail) — each surfaces a natural-language explanation referencing the rule / risk.
- [ ] **SC-007**: After feedback, reload `/api/astra/profile/user_001` — see ≥1 new signal in the updated context.

## 6. Trigger specific decision branches in the demo

The decision matrix is testable through deliberate request crafting:

| Branch (FR-012) | How to trigger |
|---|---|
| hard-rule fail → `deny` | Set `validUntil` in the past, or submit request with amount > `maxSingleAmount` |
| AI `pass` + below threshold → `allow` | Submit a small Travel request (e.g., S$45 taxi) |
| AI `pass` + at/above threshold → `ask` | Submit S$980 hotel (above typical median Travel spend) |
| AI `caution` → `ask` | Submit a quote that's > 30% above market high |
| AI `fail` + below threshold → `ask with strong warning` | Use the "non-aligned" merchant fixture (e.g., luxury jewelry stored under `Travel`) at small amount |
| AI `fail` + at/above threshold → `deny` | Same non-aligned merchant at S$980 |

## 7. `ask` expiry

Submit a request that lands in `ask`, then leave it for 15 minutes (or set `ASK_EXPIRY_SECONDS=10` in `.env.local` for fast demo). Reload — the decision shows as `expired` and the event log contains `decision.expired`.

## 8. Deploy

```bash
vercel
# Add env vars AI_GATEWAY_API_KEY and DEMO_LIVE_LLM in the Vercel dashboard
vercel --prod
```

Single Vercel project; no DB, no external services. Process restarts wipe in-memory state by design (demo property).
