Great, going with all the defaults. Here's the design.

## Summary

- **Architecture:** one small web app plus one managed Postgres database. Stripe Checkout handles cards, an email service sends QR tickets, and an offline-capable phone web app scans at the door.
- **Cost:** about $35 to $45/month plus Stripe's 2.9% + 30¢ per order.
- **Top risk:** a buyer pays after their seat hold has lapsed and the seat has gone to someone else. The design catches this and refunds automatically, but it's the case to test hardest.
- **Decisions to confirm:** the QR payload format, the ticket ID scheme, the seat ID scheme, and Stripe as the processor (details at the end).

---

## 1. Requirements and numbers

**Confirmed:** 300 reserved seats, about 60 performances a year, up to 500 buyers in the first 10 minutes of a hot on-sale, 10-minute holds, max 8 seats per order, admin comps/blocks/refunds, offline door scanning, under $50/month, one volunteer developer, nobody on call. Never double-sell a seat, and never charge someone without issuing a ticket.

| Quantity | Arithmetic | Result |
|---|---|---|
| Tickets per year | 60 performances × 300 seats | 18,000 max |
| Orders per year | 18,000 ÷ ~3 seats per order | ~6,000 |
| Average write rate | 6,000 ÷ 31.5M s/year | ~0.0002/s (effectively zero) |
| Peak seat-map reads | 500 buyers polling every 5 s | **100 reads/s** |
| Peak hold/checkout writes | ~100 orders for one sold-out show, mostly in the first 10 min = 100 ÷ 600 s ≈ 0.2/s. Allow 25× for the first-second rush | **~5 writes/s** |
| Peak Stripe webhooks | same as orders | < 1/s |
| Door scans | 300 people over ~30 min = 300 ÷ 1,800 s | ~0.2/s, on the device |
| Fan-out | One email per order. No broadcast | none |

**Storage per year**

| Data | Arithmetic | Per year |
|---|---|---|
| Orders | 6,000 × ~1 KB | 6 MB |
| Tickets | 18,000 × ~0.5 KB | 9 MB |
| Seat inventory rows | 60 × 300 × ~0.2 KB | 4 MB |
| Scan log | 18,000 × ~0.2 KB | 4 MB |
| QR images, PDFs | generated on request, not stored | 0 |
| **Total** | | **~25 MB/year** |

**Monthly cost**

| Item | Cost |
|---|---|
| App hosting (e.g. Render or Fly, small instance) | $7–$10 |
| Managed Postgres with point-in-time recovery (e.g. Neon, Supabase Pro, Render) | $19–$25 |
| Transactional email (e.g. Postmark 10k/month, or a free tier at ~2k emails in a busy month) | $0–$15 |
| Domain | ~$1 |
| Uptime and job monitoring (free tier, e.g. Healthchecks.io, UptimeRobot) | $0 |
| **Total** | **~$35–$45** |
| Stripe | 2.9% + 30¢ per order, on a $25 ticket × 3 that's ~$2.48 per order |

**Right-sizing:** 100 reads/s and 5 writes/s is a tiny fraction of what one Postgres server handles (thousands of simple queries/s). One app instance and one database are enough. No cache, no queue service, no microservices.

## 2. Architecture

```
  Buyer browser ──► Web app (one instance) ──► Postgres (managed, PITR)
     │   ▲             │   ▲      │
     │   │ redirect    │   │      └─► in-process worker: emails, hold sweep, reconciliation
     ▼   │             ▼   │ webhook
   Stripe Checkout ────────┘
                                   Email service ──► buyer's inbox (QR ticket)

  Door phones (PWA) ── download ticket list before doors ──► Web app
                    ── scan offline, sync scans when online ──►
  Admin page (same app, login required): shows, seat blocks, comps, box office, refunds, reports
```

- One codebase serves the public site, the admin page, the scanner app, and the Stripe webhook.
- The "worker" is a background loop inside the same app, with job state stored in Postgres. No separate queue.
- The seat map layout is a static JSON/SVG file in the repo. Only seat *status* comes from the database.

## 3. Data model

```
performances(id, show_id, starts_at, on_sale_at, status)
seats(id TEXT PRIMARY KEY  -- e.g. 'ORCH-C-12', section, row, number, price_tier)
seat_inventory(performance_id, seat_id, status, hold_id, hold_expires_at, ticket_id,
               PRIMARY KEY (performance_id, seat_id))
   -- status: available | held | sold | blocked
holds(id UUID, performance_id, email, created_at, expires_at, stripe_session_id UNIQUE)
orders(id UUID, hold_id UNIQUE, stripe_session_id UNIQUE, stripe_payment_intent UNIQUE,
       email, name, total_cents, status, created_at)
   -- status: paid | refunded | partially_refunded | comp
tickets(id UUID, order_id, performance_id, seat_id, status,
        UNIQUE (performance_id, seat_id) WHERE status = 'valid')
scans(ticket_id, device_id, scanned_at, result)
jobs(id, kind, payload, run_after, attempts, done_at)   -- email, sweep, reconcile
```

**Hot-path query 1, the seat map** (100/s at peak):

```sql
SELECT seat_id,
       CASE WHEN status = 'held' AND hold_expires_at < now() THEN 'available' ELSE status END
FROM seat_inventory WHERE performance_id = $1;
```

Served by the primary key `(performance_id, seat_id)`. It returns 300 small rows (~3 KB of JSON). The app can cache it for 1 second.

**Hot-path query 2, taking a hold** (atomic, no locks held across requests):

```sql
UPDATE seat_inventory
SET status = 'held', hold_id = $hold, hold_expires_at = now() + interval '10 minutes'
WHERE performance_id = $perf AND seat_id = ANY($seats)
  AND (status = 'available' OR (status = 'held' AND hold_expires_at < now()))
RETURNING seat_id;
```

If fewer rows come back than seats requested, roll back the transaction and tell the buyer "some of those seats were just taken." Postgres row locks make two simultaneous holds on the same seat impossible. The partial unique index on `tickets` is the last line of defense against a double sale.

## 4. Key flows

**Buying (hot path)**

| Step | What happens |
|---|---|
| 1 | Buyer opens the performance page. The browser loads the static seat layout and polls seat status every 5 s. |
| 2 | Buyer picks up to 8 seats and enters an email. The app runs the hold query in one transaction and creates a `holds` row. |
| 3 | App creates a Stripe Checkout Session with `client_reference_id = hold_id` and an idempotency key equal to `hold_id`. It extends the hold to match the session's expiry (Stripe's minimum session life is 30 minutes, so the hold becomes 10 min to start checkout, then up to 30 min once on Stripe's page). The buyer is redirected to Stripe. |
| 4 | Stripe sends the `checkout.session.completed` webhook. **This is the source of truth, not the browser redirect.** In one transaction the app checks that the hold still owns the seats, marks them `sold`, and inserts the order and tickets. The unique `stripe_session_id` makes repeats a no-op. It enqueues an email job. |
| 5 | The success page polls the order status and shows the tickets as soon as the webhook lands, so the buyer doesn't depend on email. |

**Background (off the hot path)**

| Job | Schedule | Purpose |
|---|---|---|
| Send ticket email | on enqueue, retry with backoff | Email with QR codes and a link to view tickets |
| Hold sweep | every minute | Expire the Stripe sessions for lapsed holds (Stripe API "expire session") so a late payment can't start. Correctness doesn't depend on this, because the seat queries already treat expired holds as available. |
| Reconciliation | every 15 min | List recent paid Stripe sessions and fix any with no order (a missed webhook) |
| Late-payment handler | in the webhook | If the hold lapsed and the seats are still free, keep the sale. If someone else took them, refund automatically and email an apology. |

**QR ticket:** the QR encodes `ticket_id` plus an HMAC signature using a server secret, e.g. `T1.<ticket_uuid>.<sig>`. It's not guessable and can't be forged, even offline.

**Door scanning:** before doors open, each phone downloads the performance's valid ticket IDs and seat labels (300 rows) and stores them in browser storage. A scan checks the signature and the list locally, marks the ticket used on that phone, and shows the seat. Scans sync to the server whenever there's signal. If two phones admit the same ticket while offline, the conflict is flagged in the sync report. That's acceptable for a 300-seat house with one or two doors.

**Admin:** block seats (status `blocked`), issue comps (an order with status `comp`, no Stripe), take box office sales (Stripe card reader or Payment Link on the same flow), refund (Stripe refund API plus ticket status `refunded`, which frees the seat), and run reports (SQL over orders).

## 5. Data protection

| Data | Cost to lose it | RPO | Mechanism |
|---|---|---|---|
| Orders and tickets | High. People paid and will show up without proof. | ~0 (minutes at most) | Managed Postgres with point-in-time recovery. Stripe holds an independent record of every payment, and reconciliation can rebuild orders from it. |
| Seat inventory / holds | Low. Rebuildable from tickets plus blocks. | 1 hour | Same database. Script to rebuild from `tickets`. |
| Scan log | Low. Only matters during the show. | Until synced | Stored on the phone until upload. The server copy is in Postgres. |
| Shows, performances, blocks, comps | Medium. Retyping is tedious. | ~0 | Same database, PITR. |
| Seat layout, code, config | Low | 0 | Git repo |
| Stripe and HMAC secrets | High. The HMAC key validates every issued QR. | n/a | The host's secret store, plus an offline copy held by two board members |

- **Availability target:** 99.5% overall (~3.6 hours down a month). During on-sale windows, aim for no downtime by having the developer watching.
- **RTO:** 1 hour. Redeploy the app, or restore the database to a point in time.
- **Door fallback:** export a printable attendee list (name, seat) for each performance an hour before curtain.

## 6. Decisions

| Choice | Alternative | Why | Hard to reverse? |
|---|---|---|---|
| Stripe Checkout (hosted) | Square, PayPal, own card form | Card data never touches you, minimal PCI scope, built-in idempotency and webhooks | **Yes.** Refunds and reconciliation for past orders are tied to Stripe IDs |
| QR payload `T1.<uuid>.<hmac>` | Plain ticket number, JWT | Short and scannable, verifiable offline, versioned with `T1` | **Yes.** Issued tickets are in people's inboxes. Changing it means the scanner must accept both formats forever, or tickets get reissued |
| Random UUID ticket and order IDs | Sequential integers | Not guessable from URLs or QR codes | **Yes.** Appears in QR codes and ticket links |
| Seat IDs like `ORCH-C-12` | Numeric IDs | Readable on tickets, at the door, and in reports | **Yes.** Printed on tickets and used in every inventory row. A layout change needs a migration |
| Ticket view URL format `/t/<order_uuid>?k=<token>` | Account login | Guest checkout, link in the email | **Yes.** Links in sent emails must keep working |
| Refund policy wording (admin-only refunds) | Self-serve | Simple, volunteers decide case by case | **Yes, if published.** It's a promise to buyers |
| Postgres with conditional updates for holds | Redis locks | One system, transactional, enough for 5 writes/s | No |
| Polling every 5 s for seat status | WebSockets | Simple, 100 req/s is trivial | No |
| In-process worker with a `jobs` table | Separate queue service | One deploy, nothing extra to run | No |
| Monolith on a small PaaS instance | Serverless, Kubernetes | Cheapest thing a single volunteer can run | No |
| Hold length 10 min (30 min on Stripe's page) | Other values | Config | No |

## 7. Failure walk

| Scenario | What happens |
|---|---|
| App crashes after Stripe charges but before the order is written | The webhook gets a non-2xx response or times out. Stripe retries for up to 3 days. The 15-minute reconciliation job also catches it. The order write is one transaction, so there's never half an order. |
| App crashes mid-hold | The hold transaction rolls back, or it committed and the hold just expires after 10 to 30 min. No seat is stuck. |
| Buyer double-clicks "Checkout" or refreshes | The checkout session is created with idempotency key `hold_id`, so they get the same session. A second hold request for seats they already hold succeeds with no change. |
| Stripe sends a webhook twice | `UNIQUE(stripe_session_id)` on orders turns the second into a no-op that still returns 200. |
| Payment completes after the hold lapsed | Seats still free: keep the sale. Seats resold: automatic refund and an apology email. Logged for the admin. |
| Two buyers grab the same seat at the same instant | Postgres row locking lets one `UPDATE` win. The other gets "seat just taken." |
| Stripe down or slow | New checkouts fail with a clear message, and held seats expire normally. No one is charged without a ticket. |
| Email service down | Email jobs retry with backoff for 24 hours. Buyers still see tickets on the success page and through the ticket link. Admin can resend. |
| No signal at the door | The scanner works from the downloaded list. The printed attendee list is the backup. |
| Phone dies at the door | A second phone with the list downloaded, or the printed list |
| Database or host outage during an on-sale | Sales pause. Nothing is half-sold, thanks to transactions. Post "on-sale delayed" on social media, restore within the 1-hour RTO. |
| **Non-code single points of failure** | |
| Domain expiry | Auto-renew on, with a card that won't expire soon. Two board members have registrar access. |
| TLS certificate | Auto-renewed by the PaaS. The uptime monitor alerts on certificate errors. |
| DNS | At the registrar or Cloudflare, with two people who have access |
| Background jobs silently stop | Each job pings a heartbeat monitor. No ping in 15 min sends an email to the developer and a board member. |
| The one volunteer developer leaves | A short runbook in the repo (deploy, restore, rotate secrets, refund). Stripe and host accounts owned by the theater, not a person. |
| Stripe webhook secret or HMAC key leaked | Rotate the webhook secret anytime. The HMAC key supports a key ID in the `T1` payload, so a new key can sign new tickets while old ones still validate. |

## 8. Not building yet

| Item | Add it when |
|---|---|
| Virtual waiting room / queue | More than 2,000 people on the site at an on-sale, or seat-map p95 latency above 1 s |
| Redis or a CDN cache for seat status | Seat-map reads above 1,000/s |
| WebSockets for live seat updates | Polling traffic above 500 req/s, or buyers complaining about stale maps (more than 5% of hold attempts fail with "just taken") |
| Separate worker process or queue service | More than 50 background jobs/s, or emails delayed more than 5 min |
| Second app instance | Uptime below 99.5% in any quarter, or CPU above 70% during an on-sale |
| Buyer accounts | More than 30% of orders from repeat buyers asking for order history, or a membership/season-ticket program |
| Apple/Google Wallet passes | More than 10 requests a season |
| Per-person purchase limits | Evidence of resellers (one email or card buying more than 16 seats for a show) |
| Multi-venue or flexible layouts | A second venue, or more than 2 layout changes a year |

## 9. Open risks

- **Hot on-sale fairness:** with 500 people and 300 seats, whoever clicks fastest wins and some people see "just taken" repeatedly. That's acceptable at this scale, but say on the site that seats are first come, first served.
- **Stripe's 30-minute session minimum** means a seat can be tied up for up to 30 minutes by someone who abandons on Stripe's page. On a sellout that delays some seats returning to sale. Expiring sessions from the sweep job reduces this.
- **Offline double admits** across two phones are possible. That's low impact for one or two doors.
- **Volunteer bus factor:** one person knows the system. The runbook and theater-owned accounts lower this but don't remove it.
- **Load testing:** do a dress rehearsal on-sale (a fake performance with 20 volunteers grabbing seats) before the first real one.

---

## Before any code: please confirm these

**The most consequential hard-to-reverse decisions:**

1. **QR payload format `T1.<ticket_uuid>.<hmac>` with a key ID.** Changing it later means the scanner must support old tickets indefinitely, or every outstanding ticket gets reissued and re-emailed.
2. **Stripe Checkout as the processor.** Switching later means past orders' refunds still have to go through Stripe, and reconciliation and webhook code get rewritten.
3. **Seat ID scheme (`ORCH-C-12`).** It's printed on tickets and keys every inventory row. Changing it needs a data migration and makes printed tickets disagree with the system.
4. **Random UUIDs for ticket and order IDs.** They're embedded in QR codes and emailed links, so changing the scheme breaks every ticket already sent.
5. **Ticket link URL format `/t/<order_uuid>?k=<token>`.** These links sit in buyers' inboxes, so a change breaks them unless the old route stays forever.

**Also hard to reverse:**
- The published refund and exchange policy (a promise to buyers).

Reply "confirmed" to lock these in, or tell me what to change.
