# System design checklist

Use this after drafting a design. Each line is a yes/no check.

## Requirements
- [ ] Functional, non-functional, and constraint requirements are all covered
- [ ] Concurrent users are known, not just total users
- [ ] Interfaces are known (web, API, mobile → push notifications)
- [ ] Safety and moderation are decided (who can delete, ban, report)

## Numbers and right-sizing
- [ ] Average, peak, read and write rates have their arithmetic shown
- [ ] Each data type is sized separately, with large media in object storage and only references in the DB
- [ ] Fan-out is counted for real-time or notification systems
- [ ] The design is no bigger than the numbers require

## Decisions
- [ ] Each choice is tied to a requirement
- [ ] Hard-to-reverse choices are flagged (ID scheme, URL format, API shape, caching semantics)
- [ ] ID generation fits the need: random when IDs must not be guessable, sortable when catching up by "after X"

## Correctness
- [ ] The datastore enforces invariants (UNIQUE/PK), with no check-then-act in app code
- [ ] Retried writes are idempotent (client-generated ID + UNIQUE)
- [ ] For real-time systems, the DB is the source of truth, push is a notification, and reconnects catch up from the last ID
- [ ] Calls to external systems choose at-most-once or at-least-once deliberately, and use idempotency keys if available

## Hot path and async
- [ ] The hot path does only what the response needs
- [ ] Core features are not coupled to nice-to-haves (analytics, email)
- [ ] Async is right-sized: in-memory batch, then DB-table queue, then dedicated queue, then log stream, each with a trigger
- [ ] Bulk jobs store per-item progress so a crash resumes instead of restarting
- [ ] External rate limits are respected, and progress is shown to the user
- [ ] Retries use backoff and capped attempts, failures are shown to someone, nothing is dropped silently
- [ ] Data read often is precomputed (rollups), and raw-data retention is decided

## Reliability and cost
- [ ] RPO and RTO are set per data type, protecting data first
- [ ] There's an availability target in nines (each nine costs ~10× more)
- [ ] Backups are restore-tested
- [ ] Single points of failure are listed, including domain, certs, DNS, and scheduled jobs
- [ ] Every "good enough for now" has a numeric upgrade trigger
- [ ] There's a monthly cost estimate in dollars
