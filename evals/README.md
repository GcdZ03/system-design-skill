# Evals

Each scenario goes to a fresh agent once **without** the skill (baseline) and once **with** it. The output is scored against the rubric.

## Scenarios

1. **Habit tracker:** "I'm starting a new side project: a habit tracker web app. Users sign up, create habits, check them off daily, and get a reminder email each evening if they haven't checked in. I'd like to launch it publicly. Can you design the system for me before I start coding?"
2. **Webhooks:** "We're a small SaaS (invoicing software, ~200 business customers). Customers want webhooks: when an invoice is paid, we POST an event to a URL they configure. Design the webhook delivery system for us — we want to start building this week."
3. **Hiking photos:** "Design a system for a photo-sharing app for a local hiking club. Members upload trail photos, comment on them, and get a feed of new photos. Starting from scratch."

## Rubric

| # | Check | Pass when |
|---|---|---|
| R1 | Asks first | Asks requirement questions (with defaults) *before* proposing a design |
| R2 | Arithmetic | Shows calculations for peak rate and storage |
| R3 | Data protection | RPO stated per data type, and irreplaceable data isn't on nightly-only backups |
| R4 | Triggers | Every deferred component has a numeric trigger |
| R5 | Non-code SPOFs | Mentions at least one of domain expiry, TLS certs, DNS, or a silently stopped scheduled job |
| R6 | Reversibility | Flags at least one hard-to-reverse decision |

## Results

- [2026-10-24 baseline (no skill)](baseline-2026-10-24.md): 1/18
- [2026-10-24 with skill, draft 1](with-skill-2026-10-24.md): 18/18
- [2026-10-24 draft 2 on fresh scenarios](draft2-2026-10-24.md): baseline ~2.5/12, draft 2 12/12 (raw outputs saved)
- [2026-10-24 draft 3, design to file](draft3-2026-10-24.md): 12/12, chat reply down from ~3,000 to ~340 words
- [2026-10-24 Codex, draft 3](codex-2026-10-24.md): 6/6, skill triggered implicitly
- [2026-10-24 Codex 0.164 retest](codex-v2-2026-10-24.md): 6/6, design doc twice as long
- [2026-10-24 Codex 0.164 baseline, no skill](codex-baseline-2026-10-24.md): ~0.5/6 on the same model
