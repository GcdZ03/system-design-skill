# Session 5: Background jobs (sending 50,000 emails)

**Problem:** an online course platform wants a "Release grades" button that emails all 50,000 students in a course.
**Facts:** the email provider allows 100 emails/s. Each API call takes ~200 ms. ~1 in 1,000 fail randomly.

**Naive design:** the web request loops over every student and calls the email API before responding "Done!"

## 1. What goes wrong

| Problem | Detail |
|---|---|
| Time | Sequential: 50k × 200 ms ≈ **2.8 hours**. At the rate limit: 50k ÷ 100/s ≈ **8 min**. The provider sets the floor. Firing everything in parallel gets you throttled. |
| No progress record | A crash at #23,000 means re-emailing 23k students or abandoning 27k. |
| Hogs the web server | Slow, non-essential work coupled to a request (Session 3 rules 10/11). |
| **Double-click (I missed this)** | The request times out after 30–60 s, the teacher clicks again, two loops run, and students get duplicate emails. |

## 2. Fixed design

```
Click "Release grades"
  → INSERT release (course_id, period)   UNIQUE   ← a second click is a no-op
  → INSERT 50,000 email_sends(release_id, student_id, status='pending')
                                          UNIQUE(release_id, student_id)
  → respond immediately: "Sending… 0 / 50,000"

Worker (separate process):
  take pending rows → send at ≤100/s → mark 'sent'
  failure → retry with backoff (1s, 10s, 1min…)
  after ~5 tries → mark 'failed', show it to the teacher
```

- **Crash:** the worker resumes from `pending` rows.
- **Double-click:** the UNIQUE constraint on the release.
- **Progress:** `count(*) WHERE status='sent'`.
- **Rate limit:** the worker paces itself.
- **Random failures:** retried automatically, never silently lost.

**Right-size:** a Postgres table is the queue. The trigger for a dedicated queue (SQS, Redis) is millions of jobs/day or many worker types.

## 3. The gap between two systems

```
1. call email API → email sent ✅
2. 💥 crash
3. (never runs) status='sent'
```

On restart, the row still says `pending`, so the email is sent again. **Saved state doesn't help**, because the provider and our database can't be updated in one transaction.

**My first answer** was "we should have saved state and continue." That misses that the saved state itself is what's stale.

| Strategy | Order | Crash in between | Name |
|---|---|---|---|
| Mark, then send | `sent` → API | never sent | at-most-once |
| Send, then mark | API → `sent` | sent twice | at-least-once |

Exactly-once isn't truly available across systems you don't control. **Pick which failure is less bad.**

**For grades, choose at-least-once.** A duplicate is mildly annoying, but a missing grade notice is worse.

**Mitigation:** an idempotency key (`release42-student123`) if the provider supports it. The crash window is milliseconds, so expect about zero duplicates per run.

## Rules for the skill (draft, continued)

20. Slow or bulk work goes to a background worker. The request records the job and returns.
21. Store per-item progress so a crash resumes instead of restarting.
22. A database table is a fine first queue. The trigger for a dedicated queue is very high volume or many job types.
23. Respect external rate limits. They set the minimum time, so show progress.
24. Retry with backoff, cap the attempts, and show failures to someone. Never drop them silently.
25. Across an external system, choose at-most-once or at-least-once deliberately. Use idempotency keys to make at-least-once safe.
