Happy to. Before I design anything I have a few questions, because the answers change the architecture, the data model, or the cost. My proposed default for each is in brackets.

1. **Volume.** How many shows a year, how many performances each, and how many people try to buy in the first few minutes of a popular on-sale? [About 8 shows a year with about 10 performances each, so roughly 80 performances and 24,000 seats a year. Up to 1,000 people on the site in the first 5 minutes of a big on-sale.]
2. **Downtime or lost data: which is worse?** [Losing data is worse. A paid order that disappears, or a seat sold twice, is the worst outcome. A few minutes of downtime is tolerable, except during an on-sale.]
3. **Budget and people.** What can you spend each month, not counting card fees, and who builds it and keeps it running? [Under $50 a month. One volunteer developer builds it, and nobody is on call.]
4. **Payments.** Is there a processor you already use or prefer? [Stripe, using its hosted Checkout page, so card numbers never touch our servers.]
5. **Seat holds.** When someone picks seats, how long are they held while that person pays? [10 minutes, then released automatically. Up to 10 seats per order.]
6. **Pricing.** Do you need price levels, discounts, comps, or season subscriptions? [2 or 3 price levels by section, promo codes, and staff-issued comp tickets. No season subscriptions in the first version.]
7. **Door scanning.** What will volunteers scan with, and is lobby internet reliable? [Volunteers use their own phones and a web page. Lobby Wi-Fi is unreliable, so the scanner must keep working offline from a list it downloads before doors open.]
8. **Box office.** Do you also sell at the door or by phone, and should those sales use the same seat map? [Yes. Staff sell through an admin page on the same seat map, so online and in-person sales can never double-book. Cash is recorded as a payment type.]
9. **Accounts, refunds, exchanges.** Do buyers need accounts, and who handles refunds and exchanges? [No accounts. Guest checkout with an email address. Staff do refunds and exchanges from the admin page.]
10. **On-sale fairness.** For a sellout, is first-come-first-served fine, or do you want a virtual waiting line? [First-come-first-served. Whoever holds seats first gets them, with no waiting room.]

**Assumptions** (tell me if any are wrong):
- One venue with one fixed 300-seat map. Staff can block seats per performance, for example for sound equipment or house seats.
- Accessible seats are marked on the map and sold like any other seat.
- Tickets are emailed with a QR code and a link to view them on a phone. No Apple or Google Wallet passes in the first version.
- Each ticket is admitted once. A forwarded ticket works for whoever arrives first.
- US-based, English only, prices include tax or tax is handled as a flat line item.
- Order and payment records are kept for 7 years for bookkeeping. Buyer emails are used only for tickets and show updates.
- The theater has an existing website, and the ticketing site lives on a subdomain like tickets.yourtheater.org.

Reply with changes, or 'defaults' to accept all.
