# Community Theater Ticketing: Design

**Summary**
- **Architecture:** one web app on a managed host, one managed Postgres with point-in-time recovery, Stripe Checkout for cards, an email provider for tickets, and an offline-capable phone scanner page.
- **Cost:** about $45/month for hosting, database, and email. Stripe charges about 2.9% + $0.30 per order on top of that.
- **Top risk:** a single app instance and database have to survive the first hour of an on-sale, and nobody is on call. Mitigation: a load test before each big on-sale, and the developer watching during it.
- **Confirm:** QR/ticket code format, the ticket link format, the seat ID scheme, the rule that "Postgres decides who gets a seat", and the 7-year retention promise.

---

## 1. Requirements and numbers

**Confirmed (defaults accepted)**

| Area | Requirement |
|---|---|
| Volume | ~8 shows/yr × ~10 performances = 80 performances, 300 seats each |
| On-sale peak | up to 1,000 people on the site in the first 5 minutes |
| Priority | No seat sold twice, no paid order lost. Short downtime OK except during an on-sale |
| Budget / people | < $50/month excluding card fees. One volunteer builds it, nobody on call |
| Payments | Stripe hosted Checkout. Card numbers never touch our servers |
| Holds | 10 minutes, max 10 seats per order |
| Pricing | 2 to 3 price zones by section, promo codes, staff comps. No subscriptions |
| Door | Volunteers' phones running a web page. Works offline |
| Box office | Admin page on the same seat map. Payment types: cash, external card, comp |
| Buyers | Guest checkout with email. Staff handle refunds and exchanges |
| Fairness | First come first served, no waiting room |

**Arithmetic**

| Quantity | Math | Result |
|---|---|---|
| Tickets per year | 80 × 300 | 24,000 |
| Orders per year | 24,000 ÷ ~2.5 seats/order | ~10,000 |
| Average order rate | 10,000 ÷ 365 | ~27/day, effectively 0/s |
| On-sale seat map reads | 1,000 users polling every 5 s → 1,000 ÷ 5 | 200 req/s to the app |
| Seat map reads hitting the DB | availability cached in memory for 1 s | ~1 query/s per performance |
| On-sale hold attempts | 1,000 users × ~3 tries in 5 min = 3,000 ÷ 300 s = 10/s; bursty ×10 | 10/s avg, ~100/s peak for a few seconds |
| Orders completed in an on-sale | 300 seats ÷ 2.5 per order, per performance | ~120 per performance |
| Stripe webhooks | 1 per order | ~120 per performance, spread over ~15 min |
| Emails | 1 per order + refunds and reminders | ~12,000/yr, up to ~1,500 in an on-sale day if all 10 performances open at once |
| Door scans | 300 per performance over ~30 min | ~0.2/s, mostly offline |

**Storage per year**

| Data | Math | Per year |
|---|---|---|
| Orders | 10,000 × ~1 KB | ~10 MB |
| Tickets | 24,000 × ~0.5 KB | ~12 MB |
| Performance seat rows | 80 × 300 × ~100 B | ~2.4 MB |
| Scan events | 24,000 × ~200 B | ~5 MB |
| Job / email log | 12,000 × ~1 KB | ~12 MB |
| **Total** | | **~40 MB/yr**, about 300 MB after 7 years |

No large media. Show images stay on the theater's existing website, and the seat map is one static SVG (~100 KB) in the code repo.

**Monthly cost** (approximate, check current prices)

| Item | $/month |
|---|---|
| App host, 1 small instance (for example Render or Fly.io) | ~$10 |
| Managed Postgres with point-in-time recovery | ~$20 |
| Transactional email (for example Postmark, 10k emails tier) | ~$15 |
| Uptime monitor, TLS, subdomain | $0 |
| **Total** | **~$45** |
| Stripe fees | 2.9% + $0.30 per order, about $1.75 on a $50 order |

**Right-sizing:** the peak is ~200 cached reads/s and ~100 single-statement writes/s for a few seconds. One app instance and one Postgres handle that easily, so the design starts there.

## 2. Architecture

```
  Buyer browser                     Volunteer phone (scanner PWA)
  seat map, polls every 5 s         ticket list cached in IndexedDB
        |                                     |  sync when online
        v                                     v
  +------------------------------------------------------+
  |  Web app (1 instance)                                |
  |  - public pages: shows, seat map, hold, checkout     |
  |  - admin pages: box office, refunds, comps, reports  |
  |  - Stripe webhook endpoint                           |
  |  - in-process worker: job queue + hold sweeper (15s) |
  +------------------------------------------------------+
        |                    |                     |
        v                    v                     v
  Postgres (managed,     Stripe Checkout       Email provider
  PITR)                  (hosted page,         (tickets, refunds)
  source of truth        webhooks back)
```

- **Postgres is the only place that decides who gets a seat.** The app never checks and then writes.
- The worker runs inside the same process and uses a jobs table in Postgres, so there is no separate queue service.
- The scanner is a page served by the same app. It runs without network once it has loaded the ticket list.

## 3. Data model

```sql
seats(            -- fixed venue map, loaded from the repo
  id text PRIMARY KEY,          -- 'ORCH-F-12'  (section-row-number)
  section text, row text, number int,
  x int, y int, price_zone text, accessible bool)

shows(id uuid PK, title, description_url)
performances(id uuid PK, show_id, starts_at, on_sale_at, status)
price_levels(performance_id, price_zone, amount_cents int, PRIMARY KEY (performance_id, price_zone))
promo_codes(code text PK, performance_id NULL, percent_off, max_uses, uses)

performance_seats(               -- inventory: one row per seat per performance
  performance_id uuid, seat_id text,
  status text CHECK (status IN ('available','held','sold','blocked')),
  hold_id uuid NULL,
  PRIMARY KEY (performance_id, seat_id))

holds(
  id uuid PK,
  client_token uuid UNIQUE,      -- idempotency key from the browser
  performance_id uuid, expires_at timestamptz,
  stripe_session_id text UNIQUE NULL,
  status text)                   -- active | converted | released

orders(
  id uuid PK,
  public_ref text UNIQUE,        -- 'T7K2-9QXM', shown in emails
  hold_id uuid UNIQUE NULL,
  stripe_session_id text UNIQUE NULL,
  email, name, total_cents int,
  channel text,                  -- online | box_office
  payment_type text,             -- stripe | cash | external_card | comp
  status text, created_at)

tickets(
  id uuid PK, order_id, performance_id, seat_id,
  code text UNIQUE,              -- 128-bit random, base32, in the QR
  status text,                   -- valid | admitted | refunded
  admitted_at timestamptz NULL)
CREATE UNIQUE INDEX one_live_ticket_per_seat
  ON tickets (performance_id, seat_id) WHERE status <> 'refunded';

scans(id uuid PK,                -- generated on the phone
  ticket_code, device_id, scanned_at, result)

jobs(id bigserial PK, kind, payload jsonb, run_at, attempts, last_error, done_at)
CREATE INDEX jobs_due ON jobs (run_at) WHERE done_at IS NULL;
```

Money is stored as integer cents. Prices are copied onto the order at purchase time.

**Hot-path queries** (both served by the `performance_seats` primary key)

```sql
-- Seat map (cached 1 s in app memory)
SELECT seat_id, status FROM performance_seats WHERE performance_id = $1;

-- Take a hold: all requested seats or none
BEGIN;
INSERT INTO holds (id, client_token, performance_id, expires_at, status)
  VALUES ($h, $token, $p, now() + interval '10 minutes', 'active')
  ON CONFLICT (client_token) DO NOTHING;      -- double-click returns the existing hold
UPDATE performance_seats
   SET status = 'held', hold_id = $h
 WHERE performance_id = $p AND seat_id = ANY($seats) AND status = 'available'
RETURNING seat_id;
-- if rows returned < number requested: ROLLBACK, tell the buyer which seats were taken
COMMIT;
```

Two buyers clicking the same seat at the same moment: Postgres row locks let exactly one UPDATE match `status = 'available'`. The other gets fewer rows back and rolls back.

## 4. Key flows

**Online purchase (hot path)**

| Step | What happens | In the response? |
|---|---|---|
| 1 | Buyer picks seats, browser sends them with a fresh `client_token` | yes |
| 2 | Hold transaction above. Response shows a 10-minute countdown | yes |
| 3 | App creates a Stripe Checkout session (Stripe idempotency key = hold ID), stores `stripe_session_id` on the hold, redirects | yes |
| 4 | Buyer pays on Stripe's page | Stripe |
| 5 | Stripe webhook `checkout.session.completed` → one transaction: insert order (`ON CONFLICT (stripe_session_id) DO NOTHING`), set seats `sold`, insert tickets, insert `send_tickets` job | webhook |
| 6 | Success page reads the order by session ID. If the webhook hasn't arrived, it asks Stripe for the session and runs step 5 itself (same idempotent code) | yes |
| 7 | Worker sends the email with the QR codes and the ticket link | background |

**Hold expiry (the sweeper, every 15 s)**

Stripe Checkout sessions can't expire sooner than 30 minutes, but our hold is 10. So a seat is never released while a payment for it could still complete:

1. Find active holds past `expires_at`.
2. No Stripe session yet: release the seats.
3. Has a session: call Stripe's "expire session" API. If Stripe says the session is already complete, run step 5 above (the buyer paid just in time). Otherwise release the seats.

This way a seat can't be sold to two people, and nobody pays for a seat that went to someone else.

**Box office sale:** same hold transaction, then the sale transaction straight away with `payment_type` cash, external card, or comp. No Stripe call. The ticket can be printed or emailed.

**Refund / exchange (staff):** a `refund` job calls the Stripe refund API with the ticket ID as idempotency key. The ticket becomes `refunded` and the seat goes back to `available`. An exchange is a refund plus a box office sale in one admin action.

**Door scanning**

| Step | What happens |
|---|---|
| Before doors | Scanner page downloads the ticket list for that performance (300 × ~60 bytes, about 20 KB) into IndexedDB |
| Scan | QR is read with the phone camera and checked against the local list: valid / already admitted / unknown. It works with no network |
| Sync | Every 10 s when online: upload scan events (phone-generated IDs, so re-uploads are harmless) and download others' scans plus new door sales |
| Fallback | Box office prints the attendee list (name, seat, order ref) before every performance |

## 5. Data protection

| Data type | Cost to lose it | RPO | Mechanism |
|---|---|---|---|
| Orders, tickets, payment records | Very high. Buyers paid, legal records for 7 years | ≤ 5 minutes | Managed Postgres point-in-time recovery (7+ days) plus a weekly logical dump to separate storage, kept 1 year. Stripe also has every payment, so lost orders can be rebuilt from it |
| Performance seat inventory | High, but can be rebuilt from tickets and blocks | ≤ 5 minutes | Same PITR. A rebuild script regenerates it from `tickets` |
| Holds | Low. They expire in 10 minutes anyway | n/a | Same PITR |
| Shows, prices, promo codes | Medium. Staff would have to re-enter them | ≤ 5 minutes | Same PITR |
| Venue seat map | Low | 0 | Lives in the git repo |
| Scan events | Low. Only matters during a performance | per performance | Also kept on each phone until synced |
| Job / email log | Low | 24 hours | Same PITR |

- **Availability target:** 99.5% overall (about 3.6 hours of downtime a month). During scheduled on-sale hours, the goal is no outage, backed by the developer watching rather than by extra servers.
- **RTO:** 1 hour normally, 15 minutes during an on-sale (restart or redeploy the app). Doors don't depend on the site because the scanner works offline.
- **Restore test:** restore PITR into a scratch database twice a year and run the inventory rebuild script against it.

## 6. Decisions

| Choice | Alternative | Why | Hard to reverse? |
|---|---|---|---|
| QR holds a random 128-bit ticket code looked up in a list | Signed token (HMAC/JWT) verified without a list | 300 tickets fit in a 20 KB offline list. No signing keys to manage or leak. Tickets can be revoked when refunded | **Yes.** Tickets already emailed or printed carry this format |
| Ticket link `tickets.<theater>.org/t/<code>` | Order-based link, or log in to view | Works from any email, nothing to remember | **Yes.** Links in buyers' inboxes must keep working |
| Seat IDs `SECTION-ROW-NUMBER` as text keys | Numeric surrogate IDs | Readable on tickets, reports, and printed lists | **Yes.** Changing the venue layout or scheme means migrating inventory and tickets |
| Postgres row locks plus a partial unique index decide seat ownership | Redis locks, or check-then-insert in app code | One system, enforced by the database, nothing extra to run | **Yes.** All inventory and sale logic is built on it |
| Inventory as one row per seat per performance | Compute availability from tickets alone | Makes the hold an atomic UPDATE and the seat map one indexed read | Somewhat. Needs a data migration to change |
| Stripe hosted Checkout | Stripe Elements on our page, or Square | Card data never touches us, so the PCI form is the short one (SAQ A). Least code | **Yes.** Past orders' refunds have to go through Stripe |
| Order ref like `T7K2-9QXM` (random, no ambiguous characters) | Sequential order numbers | Can't be guessed, easy to read over the phone | **Yes.** It's printed in buyers' emails |
| 7-year order retention, emails used only for tickets and show updates | Shorter retention, or marketing use | Bookkeeping needs it, and it keeps buyer trust | **Yes.** It's a promise to buyers in the privacy notice |
| Jobs table in Postgres, worker inside the app process | Redis or a hosted queue | ~12,000 jobs a year, so nothing else is needed | No |
| Seat map polls every 5 s with a 1 s cache | WebSockets push | 200 req/s is cheap. No connection state to manage | No |
| Hold 10 min, sweeper every 15 s, 10 seats max | | | No (config) |
| Single app instance, single managed DB | Two instances plus an HA database | Budget. Load fits on one | No |

## 7. Failure walk

| Failure | What happens | Handling |
|---|---|---|
| App crashes during the hold transaction | Transaction rolls back, no seats held | Buyer retries. The same `client_token` returns the hold if it committed |
| App crashes after Stripe payment, before the webhook is handled | Payment exists, order doesn't yet | Stripe retries webhooks for up to 3 days. The success page also fetches the session and runs the same idempotent sale code. A nightly reconcile job compares the day's Stripe payments to orders and alerts on any mismatch |
| Buyer double-clicks "Hold" or "Pay" | Two requests | `client_token` UNIQUE on holds. Stripe idempotency key = hold ID, so it's one session |
| Webhook delivered twice | Duplicate event | `orders.stripe_session_id` UNIQUE plus `ON CONFLICT DO NOTHING` |
| Buyer pays right as the hold expires | Race between the sweeper and the payment | The sweeper expires the Stripe session before releasing seats. If Stripe reports it already complete, the sale goes through |
| Two scanners admit the same forwarded ticket while both are offline | Two admissions | Accepted risk (rare). Syncing every 10 s narrows it once online. Conflicts show up in the post-show report |
| Stripe down or slow | No online sales | Holds still work. Checkout shows "payments are temporarily unavailable, your seats are held for 10 minutes". Box office can sell for cash |
| Email provider down | Tickets not sent yet | Job retries with backoff (1, 5, 15, 60 min, then hourly up to 24 h). Then it's flagged on the admin page. The buyer already saw the ticket link on the success page |
| Database down | Site down | Managed provider restarts it. Restore from PITR if needed. Scanners keep working offline |
| On-sale traffic beyond the estimate | App slow | 1 s availability cache, all-or-nothing holds. A 2,000-user load test runs before each big on-sale |
| Bots grab seats | Real fans miss out | 10-seat limit, one active hold per browser, Stripe Radar. Waiting room is listed under Not building yet |

**Single points of failure**

| SPOF | Mitigation |
|---|---|
| One app instance | Host auto-restarts it. Health check plus a free uptime monitor that alerts by text |
| One database | Managed backups and PITR. Restore tested twice a year |
| Hold sweeper (scheduled loop) silently stops | Seats would stay held forever during a sellout. Alert if any active hold is more than 2 minutes past `expires_at`. The admin page shows when the sweeper last ran |
| Nightly reconcile job stops | It writes a heartbeat row, and the monitor alerts if it's more than 26 hours old |
| Domain and DNS | Subdomain of the theater's domain. Put the domain on auto-renew with two contacts' emails |
| TLS certificate | Issued and renewed automatically by the host. The uptime monitor checks expiry |
| Stripe webhook secret / API keys | Stored in host env vars, with a copy in the theater's password manager |
| Email sender reputation | SPF, DKIM, and DMARC set up before launch. Otherwise tickets land in spam |
| The one volunteer developer | A runbook in the repo (deploy, restore, refund by hand, open a performance). Admin access for a second person |

## 8. Not building yet

| Item | Numeric trigger to add it |
|---|---|
| Virtual waiting room / queue | More than 3,000 people on the site at an on-sale, or hold requests take over 1 s at p95 in the load test |
| Redis or a CDN cache for availability | Seat map reads above 1,000 req/s |
| Second app instance, high-availability database | One outage during an on-sale, or more than $20k sold in a single on-sale hour |
| WebSocket live seat map | More than 5% of hold attempts fail because the seat was already taken (stale map) |
| Dedicated queue service | More than 50 jobs/s sustained |
| Buyer accounts / order history | More than 30% of orders come from repeat emails and people ask for it, or more than 10 "resend my tickets" requests a month |
| Apple / Google Wallet passes | More than 10% of buyers ask for it in a season survey |
| Season subscriptions | Staff decide to sell them. That needs a new order type |
| Signed offline QR codes | More than 2,000 tickets per performance (the list gets too big for phones) or multiple venues |

## 9. Open risks

| Risk | Note |
|---|---|
| On-sale hour on one instance with no on-call | Schedule on-sales when the developer is free. Run the load test the week before |
| Seat map accessibility | A clickable SVG isn't usable with a screen reader. A "list view" of available seats by section is needed at launch |
| Sweeper timing vs. Stripe's 30-minute minimum | The design handles it, but it's the trickiest code. It needs focused tests, including a payment that completes during the expire call |
| Scalpers | First come first served plus a 10-seat limit is weak against bots. Watch the first sellout |
| PCI paperwork | SAQ A is still a yearly self-assessment, and someone at the theater owns it |
| Volunteer turnover | Code, runbook, and credentials have to be handed over, not live in one person's head |
