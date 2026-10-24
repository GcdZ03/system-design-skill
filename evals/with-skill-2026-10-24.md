# With skill, 2026-10-24 (SKILL.md draft 1)

Setup: fresh Claude Code subagents (Opus 5.5), one run per scenario. Each was told to read `skills/designing-new-systems/SKILL.md` and follow it, with no other tools.

Each run had two turns:
1. The request
2. The reply "defaults"

| Check | Habit | Webhooks | Photos |
|---|---|---|---|
| R1 Asks first | ✅ 19 grouped questions with defaults, then stopped | ✅ 18 questions, then stopped | ✅ 19 questions, then stopped |
| R2 Arithmetic | ✅ ~0.4/s write peak, ~500-email reminder burst, <250 MB/yr | ✅ 2.6/s peak incl. retries, ~300 MB steady | ✅ 0.08/s upload peak, 81 GB/yr, 30 GB/mo egress |
| R3 Data protection | ✅ check-ins "irreplaceable" → PITR ≤5 min + off-site dump | ✅ per-type table, outbox in same transaction | ✅ **fixed the baseline failure**: originals copied to a 2nd provider within minutes, alert if un-backed >1h |
| R4 Triggers | ✅ every item numeric | ✅ every item numeric | ✅ every item numeric |
| R5 Non-code SPOFs | ✅ domain, TLS, DNS/email auth, cron heartbeat | ✅ silent worker and purge cron, encryption key | ✅ domain, TLS, DNS, heartbeats, expired billing card, bus factor |
| R6 Reversibility | ✅ 5 flagged | ✅ 7 flagged | ✅ 7 flagged |

**Score: 18/18**, against 1/18 for the baseline (where several checks were only partial).

## Unprompted good behavior

- The photos design noticed it was over budget ($29 vs $25) and offered a cheaper tier, naming the RPO tradeoff.
- Two designs labeled their prices as approximate and asked the user to check them.

## Issues to fix in draft 2

1. **Too many questions.** 18–19 per run is heavy, even with bulk-accept. Cap it, and ask only questions whose answers change the design.
2. **Hard-to-reverse flags are getting diluted.** The webhooks run marked 8 of 11 decisions "Yes", including minor ones. Limit the hand-off to the few most consequential decisions.
3. **Length.** The designs run ~2,500+ words. Fine as a design doc, but worth a "keep it short" check.

## Limits of this run

- n = 1 per scenario
- Claude only (Codex and Grok not tested yet)
- Same scenarios that shaped the draft, so there's a risk of overfitting. Add a fresh, unseen scenario next.
- Scored by the session that wrote the skill, not an independent grader
