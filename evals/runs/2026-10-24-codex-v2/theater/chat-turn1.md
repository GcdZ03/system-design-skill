Before designing, I need a few choices that affect cost and reliability. The [design skill](<repo>/evals/runs/2026-10-24-codex-v2/theater/.agents/skills/designing-new-systems/SKILL.md) requires: “Propose architecture, components, or a data model only after the user has answered your questions or accepted your defaults.”

1. How many performances per year, and how many buyers might arrive together when sales open? **[100 performances/year; 500 simultaneous buyers; one performance goes on sale at a time.]**
2. What is your monthly operating budget, and who will build and maintain it? **[A$150/month, excluding card fees; a volunteer developer builds it and provides occasional maintenance.]**
3. Is a brief sales outage preferable to risking lost bookings, and how quickly must service recover? **[Never knowingly lose a confirmed booking; recovery within one hour.]**
4. Can door scanners depend on internet access? **[2–4 phones, venue Wi-Fi with a mobile hotspot backup; temporary outages must still allow admission.]**
5. Do you need box-office sales, complimentary tickets, refunds, or exchanges? **[Yes to all; online and box-office sales use the same seat inventory.]**
6. What booking rules do you want? **[Guest checkout, eight-minute seat holds, six seats per order, organizer-controlled accessible seating, no customer resale.]**
7. Where do you operate, and in which currency? **[Australia; AUD.]**

**Assumptions:** One venue with 300 numbered seats; tickets belong to a specific performance; email delivery; one QR code per seat; ordinary entry is single-use with staff-controlled re-entry.

Reply with changes, or 'defaults' to accept all.