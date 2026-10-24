# Community Theater Ticketing: System Design

## 1. What actually matters here

Before the design, it helps to size the problem, because it is smaller than "ticketing system" suggests.

- **Inventory is tiny.** 300 seats per performance. If you run, say, 3 shows a year with 8 performances each, that is about 7,200 seats a year.
- **Traffic is spiky but modest.** A sell-out in an hour means 300 seats in about 150 orders (people buy 2 at a time on average). Even with 1,000 to 2,000 people refreshing the seat map at 10:00 sharp, that is a few hundred requests per second at the very peak, and only for a few minutes. A single small server and a single Postgres database handle this easily.
- **The hard problems are correctness, not scale:**
  1. Never sell the same seat twice.
  2. Never take someone's money without giving them a seat, and never give a seat without money.
  3. Let the door scan work even if the lobby Wi-Fi is bad.
  4. Keep card data off your servers entirely (PCI).
  5. Be maintainable by volunteers who will turn over. This is the constraint that should drive most choices.

So the design goal is: **one boring app, one database, a hosted payment provider, and very careful seat-locking logic.**

## 2. High-level architecture

```
  Browser (seat map, checkout)          Door scanner (phone/tablet PWA)
          |                                        |
          v                                        v
   +-------------------------------------------------------+
   |   Web app (single monolith: API + server-rendered UI) |
   |   - seat map / holds / orders / tickets / admin       |
   +-------------------------------------------------------+
          |                  |                    |
          v                  v                    v
     PostgreSQL        Stripe (Checkout       Email provider
   (source of truth)   or Payment Element,    (Postmark/SES/
                        + webhooks)            Resend)
```

- **Web app:** one deployable (Rails, Django, Laravel, or Node with something like Next.js; pick whatever the volunteers already know). Host it on a managed platform (Render, Fly.io, Railway, Heroku) so nobody has to patch servers.
- **PostgreSQL:** managed instance with automated daily backups and point-in-time recovery. This is the only stateful component. No Redis, no message queue, no microservices needed.
- **Stripe:** handles card entry, 3-D Secure, Apple/Google Pay, refunds, and receipts. Card numbers never touch your system.
- **Email:** transactional provider for ticket delivery.
- **Background jobs:** a simple DB-backed job runner (e.g. Sidekiq-free options like GoodJob, Django-Q, pg-boss) for sending emails and expiring holds. Again, no extra infrastructure.

## 3. Data model

```
venue_seats        (id, section, row, number, x, y, kind)   -- kind: standard, wheelchair, companion, obstructed
performances       (id, show_id, starts_at, on_sale_at, status)
price_tiers        (id, performance_id, name, amount_cents)  -- adult, student, senior
performance_seats  (performance_id, seat_id, price_tier_id,
                    status,            -- available | held | sold | blocked
                    hold_id NULL,
                    order_id NULL,
                    version)
                    PRIMARY KEY (performance_id, seat_id)
holds              (id, session_id, performance_id, expires_at, created_at)
orders             (id, email, name, performance_id, status,  -- pending | paid | refunded | cancelled
                    stripe_payment_intent_id UNIQUE, total_cents, created_at)
tickets            (id, order_id, performance_id, seat_id, token_id UNIQUE,
                    checked_in_at NULL, checked_in_by NULL, voided_at NULL)
                    UNIQUE (performance_id, seat_id) WHERE voided_at IS NULL
stripe_events      (event_id PRIMARY KEY, processed_at)        -- webhook idempotency
audit_log          (who, what, when, details)
```

Key idea: **one row per seat per performance** (`performance_seats`), created when the performance is set up. Seat state lives in exactly one place, and the database's row-level guarantees do the hard work.

The partial unique index on `tickets` is a second safety net: even if application logic has a bug, Postgres refuses to issue two live tickets for one seat.

## 4. The critical flow: picking and holding seats

### 4.1 Showing the seat map

- The seat layout (coordinates, labels) is static per venue: serve it as a cached JSON/SVG.
- Seat availability is a small query (`SELECT seat_id, status FROM performance_seats WHERE performance_id = ?`), 300 rows. Cache it for 1 to 2 seconds during on-sale so a thousand refreshers do not hammer the DB. A slightly stale map is fine because the **hold** step is authoritative.
- Poll every few seconds (or use Server-Sent Events) so seats turn grey as others take them. Polling is simpler and good enough.

### 4.2 Holding seats (the anti-double-booking step)

When a user clicks "Continue" with seats selected, do it atomically in one transaction:

```sql
BEGIN;
INSERT INTO holds (id, session_id, performance_id, expires_at)
  VALUES (:hold_id, :session, :perf, now() + interval '10 minutes');

UPDATE performance_seats
   SET status = 'held', hold_id = :hold_id
 WHERE performance_id = :perf
   AND seat_id = ANY(:seat_ids)
   AND (status = 'available'
        OR (status = 'held' AND hold_id IN
             (SELECT id FROM holds WHERE expires_at < now())))
RETURNING seat_id;

-- if rows returned != number requested: ROLLBACK and tell the user
-- "some of those seats were just taken", refresh the map
COMMIT;
```

Why this works: the conditional `UPDATE` takes row locks, so if two people grab seat F7 at the same instant, one update matches and the other matches zero rows. No distributed locks, no Redis. It is all-or-nothing, so nobody ends up with half their party seated.

Rules around holds:
- **Hold duration:** 10 minutes, shown as a visible countdown.
- **Limit per order:** cap at, say, 8 seats, to discourage hoarding and scalpers.
- **One active hold per session:** creating a new hold releases the old one.
- **Expiry:** treated lazily (the `UPDATE` above treats expired holds as available) plus a job every minute that flips expired holds back to `available` so the map looks right.

### 4.3 Payment

1. With seats held, the server creates an `order` (status `pending`) and a Stripe **PaymentIntent** (or Checkout Session) for the total, with `order_id` and `hold_id` in metadata. Use an idempotency key so a double-click does not create two charges.
2. The browser collects card details via Stripe's hosted fields. You never see the card number, which keeps you in the easiest PCI tier (SAQ A).
3. Stripe confirms payment and sends a **webhook** (`payment_intent.succeeded`).

### 4.4 Finalizing the order (webhook handler)

The webhook, not the browser redirect, is the source of truth (people close tabs, phones lose signal).

```
on payment_intent.succeeded(event):
  if event.id already in stripe_events: return 200        -- idempotent
  BEGIN
    lock order row
    if order already paid: record event, COMMIT, return
    UPDATE performance_seats SET status='sold', order_id=:order
      WHERE hold_id = :hold AND status = 'held'
    if all seats converted:
        mark order paid, create tickets, enqueue ticket email
    else:
        -- hold expired AND someone else took the seat (rare edge case)
        mark order 'needs_refund', enqueue automatic refund + apology email,
        alert an admin
    insert stripe_events(event.id)
  COMMIT
  return 200
```

The edge case worth thinking about: payment succeeds just after the hold expires. Mitigations, in order:
- Keep the hold a bit longer server-side than the countdown shows (e.g. show 10 minutes, actually expire at 12), and extend it when a PaymentIntent is created.
- If a seat really was lost, try to convert anyway if it is still `available` (nobody took it, so just sell it).
- Only if it was truly taken: auto-refund, and email the buyer. This should be extremely rare.

Also handle `payment_intent.payment_failed` / Checkout session expiry by releasing the hold.

## 5. Tickets and QR codes

### 5.1 What goes in the QR code

Do not just encode a ticket ID that anyone could guess. Encode a **signed token**:

```
payload = { tid: <ticket uuid>, pid: <performance id>, seat: "F7", v: 1 }
QR      = base64url(payload) + "." + base64url(Ed25519_sign(payload))
```

- **Signed with a private key** that lives only on the server. Scanners hold the public key, so they can verify a ticket is genuine **offline**.
- Random UUID ticket IDs mean nothing is guessable.
- Keep it short so the QR stays low-density and scans easily off a cracked phone screen.

### 5.2 Delivery

- Email with the QR inline, plus a link to a "My tickets" page (`/tickets/<long random token>`) that shows them again. Lost emails are the #1 support request, so also add a "resend my tickets" form keyed on email address.
- Optional: Apple/Google Wallet passes later. Not needed for v1.
- A printable PDF for patrons who want paper (common among community theater audiences, so this matters more than it might seem).

### 5.3 Door scanning

Build the scanner as a **Progressive Web App** that volunteers open on their own phones (camera-based QR scanning via a JS library). No special hardware.

- **Before doors open:** the scanner downloads the ticket list for that performance (ticket IDs, seat, name, void status) and the public key. That is 300 rows, trivial.
- **On scan:**
  1. Verify the signature locally, which rejects forgeries.
  2. Check the performance ID, which catches "wrong night" tickets, a very common real problem.
  3. Check the local list: is it voided? Already checked in?
  4. Show a big green / red screen with seat number and name.
  5. Record the check-in locally and sync to the server when online.
- **Multiple doors:** scanners sync check-ins to the server every few seconds and pull others' check-ins. If offline, a duplicate could slip through at two different doors at once. For a 300-seat house this risk is acceptable, and the server flags duplicates afterward. The front-of-house manager can resolve it in person.
- **Fallback:** a printable/searchable door list by name, for dead phones and "I deleted the email."

## 6. Handling the on-sale rush

At this scale you do not need a big virtual waiting room. You do want:

- **Load test once** before the first real on-sale (e.g. k6 simulating 2,000 users hitting the map and trying to hold seats). Confirm hold latency stays under ~200 ms.
- **Static assets on a CDN** (the platform usually does this).
- **Cache the availability endpoint for 1 to 2 seconds.**
- **Scale the app to 2 to 3 instances for the on-sale hour** and back down after. Managed platforms make this one slider.
- **Connection pooling** (PgBouncer or the platform's equivalent) so app instances do not exhaust DB connections.
- **Rate limits** on the hold endpoint per IP/session, plus a CAPTCHA (e.g. Cloudflare Turnstile) on hold creation to slow bots.
- **Optional simple queue:** if one show becomes truly huge, add a lightweight "you are in line" page that admits N sessions per minute using a signed cookie with an admission time. Build this only if you see a need.

Also: "Best available" button for people who do not want to hunt on the map. It reduces seat-map contention and is friendlier on phones.

## 7. Admin / box office features (do not skip these)

The volunteers running the theater will use these more than any patron-facing page:

- **Create performance:** pick show, date, prices; seats are generated from the venue template.
- **Block seats:** house seats, tech/sound booth, seats removed for set pieces, wheelchair spaces.
- **Comps and box office sales:** staff can issue tickets at $0 or record cash/card-at-the-window payments (Stripe Terminal or just a "paid by cash" flag).
- **Refund / exchange:** refund via Stripe API; release the seat; void the ticket so the scanner rejects it. Exchange = void old ticket, issue new one for another performance.
- **Cancel a performance:** bulk refund and email everyone.
- **Reports:** sales by performance and tier, attendance (check-ins), and a CSV export for the treasurer. Reconcile against Stripe payouts.
- **Roles:** admin, box office, door scanner. Door scanners should not be able to issue refunds.

## 8. Accessibility and patron experience

- The seat map must work with a keyboard and a screen reader (provide a list view: "Row F, seat 7, $25, available"). Legal and ethical requirement, and it also helps older patrons on small phones.
- Clearly mark wheelchair spaces and companion seats. Consider holding wheelchair spaces until a few days before the show and then releasing them.
- Mobile-first. Most buyers will be on phones.
- Show fees and the total up front.

## 9. Security and privacy

- **PCI:** Stripe hosted fields/Checkout, so card data never touches you.
- **Webhook signatures:** verify every Stripe webhook signature.
- **Admin accounts:** strong passwords plus 2FA (or just "Sign in with Google" for the volunteer group).
- **Personal data:** store only name, email, and optionally phone. Have a retention policy (e.g. delete order personal data after 2 years except what the treasurer needs).
- **Secrets:** QR signing key and Stripe keys in the hosting platform's secret store, never in the repo. Document how to rotate the QR key (old tickets would need reissuing, so do it between shows).

## 10. Reliability and operations for a volunteer team

- **Backups:** managed DB with daily backups and point-in-time restore. Practice one restore.
- **Monitoring:** uptime check on the site, error tracking (Sentry free tier), and an alert if webhooks fail. Stripe's dashboard also shows failed webhook deliveries and retries them for days, so a brief outage does not lose orders.
- **Reconciliation job:** nightly, compare paid Stripe PaymentIntents against `orders`. Anything paid in Stripe but not `paid` locally gets flagged. This catches any webhook bugs.
- **Runbook:** a one-page doc for "the site is down during on-sale," "patron says they paid but got no ticket," and "scanner will not work." Volunteers change, so write it down.
- **Test mode:** Stripe test keys in a staging environment; run a full fake on-sale before each season.

## 11. Rough cost

- Hosting (app + managed Postgres): about $20 to $50/month.
- Email: free tier or a few dollars a month.
- Stripe: about 2.9% + 30 cents per transaction (US), with nonprofit discounts sometimes available.
- Compared to typical third-party fees of $2 to $5 per ticket, savings on 7,000 tickets a year could be roughly $10k to $30k, but weigh that against volunteer development and maintenance time.

## 12. Suggested build order

1. **MVP:** performances, seat map, holds, Stripe Checkout, webhook finalization, email with signed QR, scanner PWA with offline verification, basic admin (block seats, refunds, door list).
2. **Before first big on-sale:** load test, rate limiting, CAPTCHA, reconciliation job, runbook.
3. **Later:** best-available, wallet passes, exchanges, season subscriptions, donations at checkout, waitlist for sold-out shows.

## 13. Honest caveat

Building this is very doable, and a great project, but ticketing is unforgiving on show night. Two things to consider before committing:

- Run the first show or two **in parallel** with your existing system (e.g. sell a small block of seats through the new system) before moving fully over.
- Check nonprofit-focused ticketing platforms with flat or low fees (some charge under $1 per ticket or let patrons pay the fee). If one is cheap enough, the maintenance burden saved may be worth more than the fees.

If you do build it, the pieces that deserve the most care and tests are: the atomic seat hold, the idempotent webhook handler, and the offline scanner. Everything else is ordinary web app work.
