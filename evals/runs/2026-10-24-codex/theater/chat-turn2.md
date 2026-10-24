The design uses one web application, PostgreSQL, hosted card checkout, and phone-based QR scanning.
Estimated hosting/services: **US$80–100/month, plus card-processing fees**.
Seat reservations and payment reconciliation prevent double bookings.
A controlled paper admission list handles complete internet outages.
Recovery targets need testing; this budget does not guarantee zero data loss.

Full design: [DESIGN.md](<repo>/evals/runs/2026-10-24-codex/theater/DESIGN.md). It includes capacity calculations, database structure, payment races, refunds, backups, failure handling, and launch tests.

Four consequential decisions need confirmation:

- **Permanent seat identities:** changing them later requires migrating booking history.
- **Versioned QR codes and ticket links:** changes must preserve existing tickets or reissue them.
- **Stripe hosted checkout:** switching processors requires keeping historical payment/refund support.
- **Data retention:** published promises constrain future changes; deleted records cannot be recovered.

No other hard-to-reverse decisions are proposed.

Please confirm these together, or say what to change, before implementation. This checkpoint comes from the [design skill](.agents/skills/designing-new-systems/SKILL.md), which requires confirmation “before any code is written.”