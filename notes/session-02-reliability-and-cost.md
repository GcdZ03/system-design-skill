# Session 2: Reliability and cost

**Setup:** the Session 1 link shortener runs on one server with Postgres. It dies at 3 a.m.
**Budget:** a small startup with about $100/month.

## 1. Data loss and downtime are different problems

| | Question | Name | For a shortener |
|---|---|---|---|
| **RPO** | How much data can we afford to lose (in time)? | Recovery Point Objective | Tiny. Shared links live in emails, tweets and print, and can't be updated. |
| **RTO** | How long can we be down? | Recovery Time Objective | More relaxed. Downtime is temporary. |

**Protecting data is cheap. Protecting uptime is expensive.** Protect data first.

## 2. Backup options

| Protection | What it is | RPO | Cost |
|---|---|---|---|
| Nightly backup | `pg_dump` copied to S3 | up to 24h | ~$1/mo |
| Continuous backup | WAL archiving, restore to any point in time | minutes | ~$15–30/mo on managed Postgres |
| Standby + auto failover | A second live server takes over | seconds, and covers RTO | ~2× DB cost |

**Math check:** with nightly backups, the worst case loses a day of links: 1M/month ÷ 30 ≈ **33,000 links**, each one someone's dead link.
With a 5-minute RPO, it's 33,000 ÷ 288 ≈ **115 links**, for ~$20/mo. That's an easy decision.

**An untested backup isn't a backup.** Schedule restore drills.

## 3. Availability in nines

| Target | Downtime/year | Usually takes |
|---|---|---|
| 99% | ~3.7 days | anything |
| 99.9% | ~8.8 hours | one good managed server plus a fast restore |
| 99.99% | ~52 minutes | replicas, failover, multiple zones, on-call |
| 99.999% | ~5 minutes | big-company budgets |

Each extra nine costs roughly 10× more.

**Decision:** one server + continuous backup, targeting 99.9%. That fits the $100/mo budget.

## 4. Trigger metrics: say when you'd change your mind

> A standby replica adds ~$30/mo. We get ~8.8h of downtime a year. **Upgrade when an hour of downtime costs more than ~$40**, for example once paying customers depend on their links or a contract promises uptime.

Agreeing on the upgrade trigger in advance means nobody has to argue about it later.

## 5. Single points of failure aren't only code

List everything that takes the system down if it fails alone:
- **App server:** run two behind a load balancer, or use a host that restarts it automatically.
- **DNS:** use a provider like Cloudflare.
- **Domain name:** if it expires, every link dies and someone else can buy it. Turn on auto-renew and lock the domain ($0).
- **TLS certificate:** use auto-renewal plus an expiry alert.

The cheapest reliability wins are often boring operational ones.

## Rules for the skill (draft, continued)

5. Separate data loss (RPO) from downtime (RTO), and protect data first.
6. Pick an availability target in nines. Each nine costs ~10× more.
7. Every "good enough for now" comes with a trigger metric for upgrading.
8. List single points of failure, including the non-code ones (domain, certs, DNS).
9. An untested backup isn't a backup.
