Happy to. A few answers will change the design, so before I draw anything, here are my questions. Each one has my proposed default in brackets.

1. **Volume:** How many performances per year, and how many people try to buy in the first few minutes of a popular on-sale? [About 60 performances a year (10 shows x 6 nights), up to 500 people on the site in the first 10 minutes of a hot on-sale]
2. **Worse outcome:** Which hurts more, the site being down for an hour during an on-sale, or losing a record of a sale someone paid for? [Losing a paid sale is far worse. A short outage is tolerable if no one gets charged without a ticket, and no seat gets sold twice]
3. **Budget and people:** What's the monthly budget, and who builds and maintains it? [Under $50/month plus card fees. One volunteer developer builds it, and nobody is on call]
4. **Payments:** Is a hosted processor like Stripe Checkout OK, so card numbers never touch your servers? [Yes, Stripe. That keeps you out of most PCI compliance work]
5. **Seat holds:** When someone picks seats, how long should we hold them while they pay? [10 minutes, then the seats go back on sale]
6. **Order limits:** Is there a per-order or per-person ticket cap for popular shows? [Max 8 seats per order, no per-person tracking]
7. **Box office and comps:** Do volunteers also sell at a box office or hand out comp/reserved seats (cast families, press, wheelchair spaces)? [Yes. Admins can block seats and issue comps from an admin page, and walk-up sales go through the same system]
8. **Refunds and exchanges:** Do you allow refunds, exchanges to another night, or cancelled performances? [Refunds and exchanges done by an admin only, not self-serve]
9. **Door scanning:** Is there reliable Wi-Fi or cell signal at the door, and what do scanners use? [Volunteers' phones in a browser. Signal is unreliable, so scanning must work offline and sync later]
10. **Pricing:** Are there price tiers by section, discounts (student, senior), or promo codes? [Two or three price tiers by section, plus a few discount codes]

**Assumptions** (tell me if any are wrong):
- One venue, one fixed 300-seat layout. Reserved seating only, no general admission.
- Tickets are delivered by email as a QR code (no Apple/Google Wallet passes at first).
- Buyers check out as guests with an email address, no accounts.
- No waitlist, no dynamic pricing, no resale between patrons.
- English only. Card payments only, no cash handling in the online system.
- You need sales reports per performance (seats sold, revenue) for the board, not real-time analytics.
- Buyer data is just name, email, and order details. No special compliance beyond standard privacy and Stripe's rules.

Reply with changes, or 'defaults' to accept all.
