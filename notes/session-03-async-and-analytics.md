# Session 3: Analytics without slowing the hot path

**Setup:** the link shortener from Sessions 1–2. The friend wants a dashboard of clicks per link per day.
**Load:** ~40 clicks/s average, 200–400/s peak, 100M clicks/month.

## 1. The naive design couples a nice-to-have to the core feature

```
1. SELECT long_url FROM links WHERE code = ?
2. INSERT INTO clicks (...)        ← inside the redirect
3. redirect
```

- Bursts make the INSERT slow, so redirects wait.
- If the INSERT fails (disk full, lock), redirects fail.

That's **coupling**: two things that should fail independently are tied together. Analytics going down must never take redirects down.

## 2. Do the minimum on the hot path and defer the rest

| Option | How | Good up to |
|---|---|---|
| In-memory buffer + batch insert | Redirect now, append the click to memory, flush every few seconds in one INSERT | thousands/s on one server |
| Queue (Redis, SQS) | Push the click to a queue, a worker writes it | tens of thousands/s, survives restarts |
| Log stream (Kafka) | Durable, replayable, many consumers | very large scale, many teams |

**Decision:** in-memory buffer. It's plenty for 400/s peak, and batching cuts 400 inserts/s to about 1/s.
**Tradeoff:** a crash loses the few seconds of clicks still in memory.

## 3. Different data deserves different protection

| Data | Cost to lose | RPO |
|---|---|---|
| Links | Permanent broken links for real people | minutes or better |
| Click analytics | Slightly off charts | seconds–minutes is fine |
| Billing events (future) | Money and trust | near zero → durable queue |

Analytics are approximate by nature, so losing a few seconds is invisible. **Trigger:** if clicks ever become billing data, upgrade to a durable queue.

## 4. The dashboard query: work at write time or read time?

At 1.2B raw rows a year, `count(*) ... GROUP BY day` per page view is too slow.

- **Index `(code, clicked_at)`:** skips the full scan, but a viral link still counts millions of rows on every load.
- **Rollup table:** `link_daily_clicks(code, day, count)` with PK `(code, day)`. The batch flush adds to `count`. The dashboard reads about 365 rows per link per year.

**Precompute when the same question is asked again and again.**

**Retention:** keep raw clicks ~90 days (for bug fixes and new metrics), then archive them to S3 or delete them. Keep rollups forever.

## Rules for the skill (draft, continued)

10. Keep the hot path minimal. Anything not needed for the response happens afterwards.
11. Don't couple a core feature to a nice-to-have. Each should be able to fail alone.
12. Right-size async: in-memory batch, then queue, then log stream, each with an upgrade trigger.
13. Classify data by what it costs to lose it, and give each class its own RPO.
14. Precompute what's read often, and decide raw data retention upfront.
