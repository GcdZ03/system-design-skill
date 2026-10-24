# Session 1: Link shortener

**Problem:** "Build a link shortener like bit.ly."
**Friend's answers:** public, about 1M new links a month, about 100× that in clicks, custom aliases yes, analytics later.

## 1. Requirements come before components

There are three kinds of question to cover before picking any technology.

| Kind | Asks | Example |
|---|---|---|
| Functional | What does it *do*? | Custom aliases? Expiry? Click analytics? |
| Non-functional | How *well*? Scale, speed, uptime | How many users? Read vs write ratio? |
| Constraints | What's already decided? | Existing stack, budget, team skills |

My first questions covered scale and constraints but missed functional scope. Each feature adds real work, and analytics alone can be bigger than the shortener.

Ask about **interfaces** (web, API, browser extension), not hardware, unless the system really is tied to a device.

## 2. Estimate, then right-size

Shortcut: **1M per month ≈ 0.4 per second** (a month is about 2.6M seconds).

| Number | Value |
|---|---|
| Average reads | 100M/month ≈ 40/s |
| Peak reads (2–10× average) | about 200–400/s |
| Writes | 1M/month ≈ 0.4/s |
| Storage | 500 B × 1M/month ≈ 6 GB/year |

**Verdict:** one server running Postgres is plenty. Kafka, sharding and microservices would be over-engineering.

**The math tells you what *not* to build.**

## 3. Hot path and data model

The hot path is: look up `code` → redirect to `long_url`.

```
links(code TEXT PRIMARY KEY, long_url TEXT NOT NULL, owner_id, created_at)
```

The primary key is indexed, so a lookup takes about a millisecond. Write the data model early, because the table shape shows you which lookups are fast.

## 4. Watch for hard-to-reverse choices

301 vs 302 redirects. A 301 is cached by browsers forever, so you lose analytics and can't take it back for links already shared. **Choose 302** because the requirements say "analytics later" and we have headroom for the extra load.

For every choice, ask:
- Which requirement does it serve?
- Does it close off a future one?
- Is it cheap or expensive to reverse? (Redirect type, URL format and ID scheme are expensive. Library choices are cheap.)

## 5. Generating IDs: let the database enforce invariants

- Use random base62 codes. 6 chars ≈ 57B combinations, 7 chars ≈ 3.5T.
- At 12M links a year, a collision happens less than 0.1% of the time, so just retry.
- **Bug in "check then insert":** two requests can both see the code as free, which is a race condition.
- **Fix:** insert and rely on the PRIMARY KEY. On a duplicate key error, retry with a new code, or tell the user "alias taken."

Alternatives:
- **Counter + encode:** no collisions, but anyone can guess every link by counting up, which leaks privacy.
- **Hash of the URL:** different users share one code and its analytics.

## Rules for the skill (draft)

1. Cover functional, non-functional and constraint requirements before naming components.
2. Estimate average, peak, writes and storage, then right-size. Let the numbers argue against complexity.
3. Tie every choice to a requirement, and flag choices that are hard to reverse.
4. Let the datastore enforce invariants. Don't check-then-act.
