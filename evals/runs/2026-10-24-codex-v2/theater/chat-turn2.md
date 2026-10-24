Use one web application, managed PostgreSQL, a small worker, Stripe payments, and QR scanning.  
Eight-minute seat holds and database transactions prevent double-booking.  
Estimated running cost: **A$90/month**, with an **A$120 allowance**, excluding card fees and development.  
Offline admission uses one designated scanner, with limited duplicate detection.  
Whole-region disasters have weaker recovery guarantees than ordinary server failures.

The full design—including architecture diagram, capacity calculations, payment flows, backups, and failure handling—is saved in [DESIGN.md](<repo>/evals/runs/2026-10-24-codex-v2/theater/DESIGN.md).

The consequential decisions to confirm are:

- **Stable seat identities and historical layouts:** changing them requires migrating bookings while preserving old tickets.
- **Signed, versioned QR codes:** changing the format requires supporting tickets already printed.
- **Stripe with separate authorization and capture:** switching processors leaves historical refunds tied to Stripe.
- **Provisional offline admission:** stronger guarantees require local coordination hardware or connectivity.
- **Retention promises:** deleted history cannot be recovered; published privacy promises constrain future changes.

Other lasting decisions: separate guest-access credentials and private booking pages excluded from shared caches.

The [design skill](<repo>/evals/runs/2026-10-24-codex-v2/theater/.agents/skills/designing-new-systems/SKILL.md) requires confirmation “before any code is written.” Please confirm these choices together, or say what to change.