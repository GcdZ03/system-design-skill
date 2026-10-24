**Summary**
- **Architecture:** one web app on a managed host, one managed Postgres with point-in-time recovery, Stripe Checkout for cards, an email provider for tickets, and an offline-capable phone scanner page.
- **Cost:** about $45/month for hosting, database, and email. Stripe charges about 2.9% + $0.30 per order on top of that.
- **Top risk:** a single app instance and database have to survive the first hour of an on-sale, and nobody is on call. Mitigation: a load test before each big on-sale, and the developer watching during it.
- **Confirm:** QR/ticket code format, the ticket link format, the seat ID scheme, the rule that "Postgres decides who gets a seat", and the 7-year retention promise.

The full design is saved at `<repo>/evals/runs/2026-10-24-draft3/theater/DESIGN.md`.

**Most consequential hard-to-reverse decisions**

1. **The QR code holds a random 128-bit ticket code, and scanners check it against a downloaded list** (not a signed token). Changing it later means every ticket already emailed or printed stays in the old format, so the scanner has to support both for at least a season.
2. **Postgres decides who gets each seat**: an all-or-nothing UPDATE on per-seat rows, plus a unique index so a seat can only have one live ticket. Replacing it later means rewriting the hold, sale, refund, and box office code, and that is where double-selling bugs come from.
3. **Seat IDs are readable text like `ORCH-F-12`.** Changing the scheme or the venue layout later means migrating every inventory row and ticket, and reprinting attendee lists.
4. **Stripe hosted Checkout handles payments.** Switching processors later means past orders still have to be refunded through Stripe, so you'd run two processors side by side for months.
5. **Ticket links look like `tickets.<theater>.org/t/<code>`.** They're in buyers' inboxes, so changing them later means keeping redirects working forever.

**Other hard-to-reverse decisions**
- One inventory row per seat per performance (changing it needs a data migration)
- Order references like `T7K2-9QXM`, printed in emails
- Order records kept for 7 years, and buyer emails used only for tickets and show updates (a promise in the privacy notice)

Please confirm these in one reply, or tell me what to change, before any code is written.
