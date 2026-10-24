# Session 4: Real-time chat app

**Problem:** "Build our own chat app, like a simple Slack," for a university student community. Members join channels, send messages, and see new messages live.

## 1. Requirements (I led this time)

I covered all three kinds without prompting: core features, file uploads, history, how fast "live" must be, devices, and existing systems.

**Gaps:**
- **Scale numbers.** For chat, ask about *concurrent online users*, not just total members. Open connections cost resources even when idle.
- **Safety and moderation.** Who can delete messages, ban people, or report harassment? It's painful to add later, which makes it a hard-to-reverse choice.

Devices matter here, because mobile means push notifications, which is a whole extra system.

**Friend's answers:** 5,000 members, ~1,000 online at peak, ~50k messages/day (~200 B each), images up to 10 MB, history kept forever, under 1 s delivery, web first, moving off Discord to own their data.

## 2. Estimation

| Number | Value |
|---|---|
| Average messages | 50k/day ÷ 86,400 ≈ **0.6/s** |
| Peak (chat is bursty, ~10×) | **~6/s** |
| Text storage | 50k × 200 B = 10 MB/day ≈ **3.65 GB/year** |
| Fan-out at peak | 6 msg/s × 1,000 viewers ≈ **6,000 deliveries/s** |
| Images (5% of msgs, 2 MB avg) | 5 GB/day ≈ **1.8 TB/year**, ~500× the text |

**My mistake:** I divided a *daily* count by a *month's* seconds and got 0.02/s.
**Sanity check:** a day is ~86,400 s, so call it 100k. 50k/day is about 0.5/s.

**Lessons:**
- Delivery (fan-out) is the real load, not sending.
- Images go in object storage (S3, R2) behind a CDN, and the database stores only the URL.

## 3. Real-time delivery

| Approach | How | Fit for chat |
|---|---|---|
| Polling | Browser asks every few seconds | Slow and wasteful |
| Server-Sent Events | Server pushes, one direction | OK for feeds |
| **WebSockets** | Two-way, always open | ✅ Also carries typing and presence |

- Keep connections healthy with **heartbeats** (a ping every ~30 s) so dead ones get noticed.
- One server holds 1,000 connections easily.
- **Trigger:** once there's a second chat server, add pub/sub (Redis) so a message posted on server A reaches users on server B.

## 4. Reconnects: the database is the source of truth

Connections drop all the time (sleep, tunnels, Wi-Fi changes), and a WebSocket only delivers while connected.

> The database is the source of truth. The WebSocket is only a fast notification on top of it.

**On reconnect:** the client sends its last message ID, and the server returns `WHERE channel_id = ? AND id > ?`.

This requires **ordered IDs**, which is the opposite of the random link codes from Session 1. Same question, different answer, because the requirement is different.
- **Auto-increment (`bigserial`):** right-sized for one database.
- **ULID or Snowflake:** the trigger is many servers generating IDs without coordinating.

**The server assigns order and time.** Client clocks are wrong.

## 5. Duplicates and idempotency

- **The same message arrives by WebSocket and by catch-up fetch:** the client drops duplicates by ID.
- **The send is retried after a dropped connection:** the client attaches a `client_msg_id`, and a UNIQUE constraint makes the retry return the original message instead of posting it twice. The send is now **idempotent**: doing it twice equals doing it once.

## Rules that carried over from the link shortener

| Rule | How it showed up in chat |
|---|---|
| 1. Three kinds of requirement | Done unprompted, but scale numbers were missed |
| 2. Estimate, then right-size | One server, no Redis until there's a second server |
| 3. Choices serve requirements | Random IDs for links, sortable IDs for messages |
| 4. Database enforces invariants | Idempotent sends through a UNIQUE constraint |
| 7. Trigger metrics | Pub/sub at the 2nd server, Snowflake IDs with many writers |

## Rules for the skill (draft, continued)

15. Estimate each kind of data separately. Large media is often orders of magnitude bigger than the text.
16. Large files go in object storage, and the database keeps only references.
17. For real-time systems, measure fan-out (deliveries), not just sends.
18. The database is the source of truth, and real-time push is a notification on top. Design for reconnects.
19. Make retried writes idempotent with a client-generated ID and a UNIQUE constraint.
