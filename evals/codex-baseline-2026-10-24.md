# Codex baseline, 2026-10-24 (no skill)

**Setup**
- Codex CLI v0.164.0, model `gpt-6.1-sol`, reasoning effort high.
- These are the same model and settings as the [0.164 retest with the skill](codex-v2-2026-10-24.md).
- It ran in an empty folder outside this repo. No user-level skills were installed, only Codex's bundled `.system` skills.
- Single turn, theater scenario. Codex answered in full without asking anything, so no second turn was needed.

The raw output and log are in [`runs/2026-10-24-codex-baseline/theater/`](runs/2026-10-24-codex-baseline/theater/).

## Scores

| Check | Without skill | With skill (same model) |
|---|---|---|
| R1 Asks first | ❌ stated 3 assumptions and designed | ✅ 7 questions with defaults |
| R2 Arithmetic | ⚠️ one illustrative figure ("1,000 browsers polling every five seconds ≈ 200 req/s"). No storage or cost | ✅ per-operation table, storage, monthly cost |
| R3 Data protection per type | ❌ "enable managed backups", no RPO | ✅ RPO table, 99.9% target, hourly off-account snapshot |
| R4 Numeric triggers | ❌ "add a waiting room if measured demand exceeds tested capacity" | ✅ 9 numeric triggers |
| R5 Non-code SPOFs | ❌ none (domain, TLS, DNS and silent jobs not mentioned) | ✅ domain/DNS/TLS/billing, job heartbeat |
| R6 Reversibility | ❌ none flagged | ✅ 5 named + 2 others |
| **Score** | **~0.5 / 6** | **6 / 6** |

## What the baseline did well

The baseline was strong on correctness:
- all-or-nothing seat locking in Postgres
- never releasing seats while a payment might still succeed
- idempotent webhooks, plus a reconciliation job
- hashed QR tokens, plus atomic admission across doors

That matches the Claude baselines: the knowledge is there, and the process discipline is what's missing.

It was also much shorter (1,167 words in chat, no file), and it cited Stripe and Postgres docs from web searches.

## Limits

- n = 1.
- Scored by the session that wrote the skill.
