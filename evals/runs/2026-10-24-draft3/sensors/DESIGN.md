# Greenhouse Sensor Monitoring: Design

**Summary**
- **Architecture:** sensors publish over MQTT/TLS to one cloud VM (Mosquitto, Postgres + TimescaleDB, and an app split into three processes: ingest, alerter, web), with SMS through Twilio and an external dead-man's switch watching the alerter.
- **Cost:** about $35/month, plus about $150–300 one-time per site for an independent cellular temperature alarm.
- **Top risk:** if a site's internet drops, the cloud can't see the heat. You get a "site offline" text within 5 minutes, but no temperature alert. The cellular alarm covers that gap.
- **Confirm:** the MQTT topic/payload contract flashed into the firmware, hardware-MAC sensor IDs, 90-day raw retention, and storing each reading's greenhouse at write time.

---

## 1. Requirements and numbers

**Confirmed (defaults accepted):** 400 sensors, a reading every 10 s, 4 greenhouses at 2 sites. Raw readings kept 90 days, 1-minute rollups kept forever. A missed alert is the worst failure. Losing a few minutes of readings or an hour of dashboard is fine. Budget under $50/month, run by one developer. One broadband line per site. ESP32 boards you flash yourself. The alert fires on a per-greenhouse min/max on the 5-minute median temperature, or after 5 minutes of silence. It texts up to 4 numbers and repeats every 15 min until someone replies ACK or the reading recovers. Up to 5 dashboard users, with 10–30 s refresh.

| Quantity | Arithmetic | Result |
|---|---|---|
| Write rate, average | 400 sensors ÷ 10 s | **40 readings/s** |
| Write rate, peak | 10× for reconnect bursts after a Wi-Fi/AP blip | **400 readings/s** (ingest batches these into about 1 insert/s) |
| Readings per day | 40 × 86,400 | **3.46M/day** |
| Readings per year | 3.46M × 365 | **1.26B/year** |
| Dashboard reads | 5 users × ~4 chart queries ÷ 15 s refresh | **~1.3 queries/s** (peak 10×, about 13/s) |
| Alert evaluation | 4 greenhouses, every 30 s, in memory | negligible |
| SMS per incident | 4 numbers × 4/hour × up to 8 h unacknowledged | **≤128 texts**, about $1.50 |
| Raw storage | ~60 B/row (tuple + index) × 3.46M/day × 90 days | **~19 GB uncompressed**, ~2 GB with Timescale compression (~10×) |
| Rollup storage | 400 × 1,440 min/day × 365 = 210M rows/year × ~60 B | **~13 GB/year uncompressed**, ~1.5 GB/year compressed |
| Config, incidents | thousands of rows | < 10 MB |

**Right-sizing:** 40 writes/s and about 1 read/s is far below what one Postgres server handles (thousands of simple queries/s). The ingest batch of about 40 rows/s is trivial. One VM is enough with a big margin.

**Monthly cost**

| Item | $/month |
|---|---|
| VM, 2 vCPU / 4 GB / 80 GB SSD (e.g. DigitalOcean or Hetzner) | ~$24 (Hetzner ~$9) |
| Provider snapshots | ~$5 |
| Off-site WAL/base backups (Backblaze B2 / S3, ~20 GB) | ~$1 |
| Twilio number + US A2P 10DLC registration + texts | ~$5 |
| Healthchecks.io (dead-man's switch) | $0 (free tier) |
| Domain | ~$1 |
| **Total** | **~$35** |
| One-time: standalone cellular temperature alarm, 1 per site | $150–300 each, plus a small monthly SIM fee |

## 2. Architecture

```
 Greenhouse site (x2)                         Cloud VM (single, Docker Compose)
 ┌──────────────────────┐                     ┌───────────────────────────────────────────┐
 │ ESP32 sensors (~200) │  MQTT over TLS      │ Caddy (HTTPS for the web only)            │
 │   ── Wi-Fi APs ──────┼────── :8883 ───────▶│ Mosquitto (own CA, per-device creds)      │
 │ Router / broadband   │                     │   │            │                          │
 └──────────────────────┘                     │   ▼            ▼                          │
 ┌──────────────────────┐                     │ ingest      alerter ──▶ Twilio SMS        │
 │ Cellular temp alarm  │── own SMS ──▶ phones│   │  (in-memory 5-min windows)            │
 │ (independent backup) │                     │   │            │ ping every 60 s           │
 └──────────────────────┘                     │   ▼            ▼                          │
                                              │ Postgres + TimescaleDB   ──▶ Healthchecks.io
                                              │   ▲   (WAL to B2)            (texts/emails if
                                              │   │                           pings stop)
                                              │ web (dashboard, thresholds, Twilio ACK hook)
                                              └───────────────────────────────────────────┘
```

| Component | Role |
|---|---|
| ESP32 firmware | Reads sensors every 10 s and publishes QoS 1. Buffers up to ~10 min in RAM during a disconnect. OTA updates enabled from day 1. |
| Mosquitto | MQTT broker on the VM. TLS uses your own long-lived CA (not Let's Encrypt), so a public cert rotation can't break 400 devices. |
| ingest | Subscribes to `gh/#` and batch-inserts once a second with `ON CONFLICT DO NOTHING`. |
| alerter | Subscribes to MQTT directly, so alerts don't depend on ingest or the DB being healthy. Keeps a 5-min window per greenhouse, evaluates every 30 s, sends SMS, and pings Healthchecks.io. |
| web | Server-rendered dashboard plus small chart JS that polls every 15 s. Covers threshold editing, sensor placement, logins, and the Twilio inbound-SMS webhook for ACK. |
| Postgres + TimescaleDB | Readings hypertable, continuous 1-min aggregate, compression after 7 days, raw retention 90 days. |
| Cellular alarm | Off-the-shelf unit per site with its own thermometer and SIM. It's the only thing that works when the site's internet is down. |

## 3. Data model

```sql
site(id smallint PK, name, timezone)
greenhouse(id smallint PK, site_id, name,
           temp_min_c real, temp_max_c real, hysteresis_c real DEFAULT 1.0)
sensor(id macaddr PK,            -- burned-in ESP32 MAC, also the MQTT client id
       metric text CHECK (metric IN ('temp_c','rh_pct','soil_pct')),
       greenhouse_id smallint, label text, active bool)
recipient(id, name, phone_e164, active)
app_user(id, email, password_hash)

reading(                         -- Timescale hypertable, 1-day chunks
  ts            timestamptz NOT NULL,   -- device time, replaced by server time if skew > 60 s
  sensor_id     macaddr     NOT NULL,
  greenhouse_id smallint    NOT NULL,   -- copied at ingest, so history stays right when sensors move
  value         real        NOT NULL,
  PRIMARY KEY (sensor_id, ts)           -- dedups QoS 1 redeliveries
)
reading_1m  -- continuous aggregate: bucket, sensor_id, greenhouse_id, avg, min, max, n

incident(id, greenhouse_id, kind ('too_hot','too_cold','silent','site_offline'),
         opened_at, last_notified_at, acked_by, acked_at, closed_at)
  UNIQUE (greenhouse_id, kind) WHERE closed_at IS NULL   -- one open incident per kind
notification(id, incident_id, recipient_id, sent_at, provider_sid, status)
```

**Hot paths and their indexes**

| Query | SQL shape | Served by |
|---|---|---|
| Alert decision | Done in memory in the alerter (median of the last 5 min of `temp_c` per greenhouse). No DB on this path. | n/a |
| Live chart (last 6 h, one greenhouse) | `SELECT bucket, sensor_id, avg FROM reading_1m WHERE greenhouse_id=$1 AND bucket > now()-'6h' ORDER BY bucket` | index `reading_1m (greenhouse_id, bucket DESC)` |
| Latest value per sensor (tiles) | `SELECT DISTINCT ON (sensor_id) ... FROM reading WHERE greenhouse_id=$1 AND ts > now()-'2 min'` | index `reading (greenhouse_id, ts DESC)` |
| Historical chart (months) | Query `reading_1m`, or an hourly rollup made from it, across the range | same index on `reading_1m` |

Store all values in SI units (°C, %) and convert to °F only in the UI.

## 4. Key flows

**Hot path 1: reading to alert (must not fail)**
1. The sensor publishes `gh/{site}/{greenhouse}/{mac}` with payload `{"v":1,"ts":1729760000,"seq":812,"temp_c":31.4}`, QoS 1.
2. The alerter receives it and appends it to that greenhouse's 5-min window. It also records last-seen per sensor and per site.
3. Every 30 s, per greenhouse, it applies these rules:
   - If `median(temp_c) > temp_max_c` or `< temp_min_c` and no open incident exists, it opens one and texts all recipients.
   - It closes the incident when the median is back inside the range by `hysteresis_c`, and sends a single "recovered" text.
   - If there are no readings for 5 min from the greenhouse, it opens a `silent` incident. If every greenhouse at a site is silent at once, it opens one `site_offline` incident instead.
4. While an incident is open and not acked, it re-texts every 15 min.
5. After each loop it pings Healthchecks.io. If those pings stop for 3 min, Healthchecks notifies you on its own channels.
6. For thresholds, the alerter reads `greenhouse` from the DB every 60 s and keeps the last good copy if the DB is down. Incident writes go to the DB. If the DB is down, it keeps the incident state in memory and texts anyway.

**Hot path 2: dashboard.** The browser polls `/api/greenhouse/{id}/series?range=6h` every 15 s and gets back JSON from `reading_1m`. There are no websockets.

**ACK.** A recipient replies "ACK" by SMS. The Twilio webhook hits `/twilio/inbound`, which verifies the signature, maps the phone number to a recipient, and sets `acked_by/acked_at` on that greenhouse's open incidents. The alerter stops repeating, and everyone gets "Acked by Mom".

**Background work (not on the response path):** ingest batching, Timescale compression after 7 days, dropping raw chunks older than 90 days, continuous-aggregate refresh every minute, and the nightly base backup, which pings its own Healthchecks check.

## 5. Data protection

| Data type | Cost to lose it | RPO | Mechanism |
|---|---|---|---|
| Config: thresholds, sensor placements, recipients, users | High. Hand-entered, and wrong thresholds mean missed alerts | ≤ 1 min | Continuous WAL archiving (wal-g → B2, `archive_timeout=60s`) + nightly base backup = PITR. Also a nightly `pg_dump` of config tables |
| 1-min rollups (kept forever) | Medium-high. Seasonal history can't be recreated | ≤ 1 min | Same WAL/PITR |
| Raw readings (90 days) | Low. A gap in the charts | ≤ 1 min in the DB. Minutes during a VM outage are accepted | Same WAL/PITR. Sensors buffer ~10 min in RAM |
| Incidents and notification log | Low-medium. Useful for review | ≤ 1 min | Same WAL/PITR |
| In-flight alert state | High during an incident | 0 for the alert itself | The alerter texts first and records after. Healthchecks catches an alerter that's dead |

| Target | Value |
|---|---|
| Alerting availability | **99.9%** (≤ 43 min/month without alert coverage). The cellular alarms add an independent layer |
| Dashboard availability | **99%** (≤ 7.3 h/month) |
| RTO, alerting | **1 hour.** Rebuild the VM from a provisioning script + Compose file, restore from B2, and repoint DNS |
| RTO, dashboard/history | **4 hours** |

Test restores once a quarter into a scratch VM.

## 6. Decisions

| Choice | Alternative | Why | Hard to reverse? |
|---|---|---|---|
| **MQTT topic + versioned JSON payload in firmware** (`gh/{site}/{gh}/{mac}`, `{"v":1,...}`) | HTTPS POST per reading | Persistent connection, QoS 1 retries, and cheap on the ESP32. The `v` field allows new shapes later | **Yes.** It's baked into 400 devices. Changing it means OTA to every device, and any device without working OTA has to be reflashed by hand |
| **Sensor ID = hardware MAC** | Assigned integer IDs | No provisioning step, unique, and printed on the board | **Yes.** It's in every reading row and every device credential |
| **Raw retention 90 days, 1-min rollups forever** | Keep raw forever (~19 GB/year compressed-ish) | Keeps the disk and cost flat | **Yes.** Dropped raw data can't be recovered |
| **`greenhouse_id` stored on each reading** | Join to the sensor's current placement | History stays right when a sensor moves | **Yes.** Adding it later means a backfill over 1B+ rows, with a guess at past placements |
| **TimescaleDB on Postgres** | Plain Postgres with partitioning, or InfluxDB | Compression, retention, and continuous aggregates built in, still plain SQL | **Yes.** Leaving it means migrating every hypertable |
| SI units in storage | Store °F | One unit, no ambiguity | Yes, but cheap now. Mixed units later would need a data migration |
| Own CA for device TLS | Let's Encrypt on MQTT | Public chain rotations can't brick devices | Yes. The CA is in firmware, so pick a 20-year root |
| Alerter reads MQTT directly | Alerter queries the DB | Alerts survive ingest or DB failure | No |
| Single VM, Docker Compose | Managed DB + PaaS | ~$35 vs ~$100+. 40 writes/s doesn't need more | No |
| Dashboard polling every 15 s | WebSockets | Simpler, and fine for 5 users | No (config) |
| Twilio SMS | Push app, email | SMS works on every phone and at night | No. The sender sits behind one function |
| Thresholds, repeat interval, window, hysteresis | n/a | n/a | No (config) |

## 7. Failure walk

| Scenario | What happens | Handling |
|---|---|---|
| **ingest crashes mid-batch** | Up to ~1 s of rows not yet committed | QoS 1 persistent session redelivers anything not acked. `PRIMARY KEY (sensor_id, ts)` + `ON CONFLICT DO NOTHING` drops duplicates. Docker `restart: always` |
| **alerter crashes mid-incident** | The window is lost and pending texts may not go out | On restart it reloads open incidents from the DB and rebuilds windows. It holds off "silent" alerts for the first 5 min after startup. Healthchecks fires if it stays down for 3 min or more |
| **alerter crashes after Twilio accepted a text but before recording it** | A recipient may get a duplicate | Accepted. A duplicate text is better than a missing one |
| **Sensor double-publishes (QoS 1)** | Duplicate reading | Dedup by primary key. The alert window dedups on `(mac, ts)` |
| **Double ACK / ACK with no open incident** | n/a | ACK is an idempotent update. With no open incident, the reply is "nothing open" |
| **Twilio down or slow** | Texts fail | 3 retries with backoff (10 s, 30 s, 90 s), then a fallback email via a second provider. The alerter also skips its healthcheck ping, so Healthchecks notifies through its own channels. Twilio auto-recharge stays on so the balance can't run out |
| **Site internet or Wi-Fi down** | The cloud sees silence, not temperature | A `site_offline` text arrives within 5 min. The cellular alarm keeps watching temperature. Sensors buffer ~10 min |
| **Site power outage** | Sensors and heaters both fail | Shows up as `site_offline`, which is exactly when you want to know |
| **DB down or disk full** | No new chart data | The alerter keeps working from MQTT with cached thresholds. A disk check above 80% pings a separate Healthchecks check |
| **Clock skew on a device** | Readings land at the wrong time | Ingest replaces `ts` with server time if skew is over 60 s and counts these per sensor |
| **One sensor fails high (e.g. 85 °C)** | Could cause a false alarm | Using the median of ~25 temp sensors per greenhouse makes one bad sensor harmless. The dashboard flags outliers |
| **SPOF: the VM** | All cloud alerting stops | Healthchecks.io notices the missing pings. The cellular alarms keep working. RTO 1 h via script |
| **SPOF: domain expiry** | Devices and dashboard can't resolve | Auto-renew with a card that won't expire soon, a 5-year registration, and two contact emails. Devices could use a fixed IP as a fallback, but that's not built yet |
| **SPOF: TLS certs** | Web cert: Caddy auto-renews. Device CA: your own 20-year root | Healthchecks or an uptime check on HTTPS flags expiry within 14 days |
| **SPOF: DNS** | Same as domain | Registrar DNS with low churn. Record the VM IP in the runbook |
| **SPOF: scheduled jobs silently stop** (backups, Timescale policies, aggregate refresh) | Backups go stale, or the disk fills | Each job pings its own Healthchecks check. A daily check that `max(bucket)` in `reading_1m` is less than 5 min old |
| **SPOF: 10DLC registration revoked or carrier filtering** | Texts silently dropped | Watch Twilio delivery status. Undelivered texts trigger the email fallback |
| **SPOF: you** | Nobody knows how to fix it | One-page runbook: restore steps, credentials location, how to silence alerts, how the cellular alarm works |

## 8. Not building yet

| Item | Numeric trigger to add it |
|---|---|
| On-site edge gateway with local alerting + cellular uplink | A site's internet goes down more than once a month, or any outage lasts over 30 min |
| Second VM / hot standby for the alerter | Alerting misses 99.9% (over 43 min down) in 2 months of any 6 |
| Kafka or another durable queue in front of ingest | Sustained writes above 5,000 readings/s (e.g. 10× the sensors at a 2 s interval) |
| Separate DB server or read replica | DB CPU above 60% for an hour, or dashboard p95 above 1 s |
| WebSockets / server push for live charts | More than 50 concurrent viewers, or a need for under 5 s latency |
| Alerts on humidity or soil moisture | The first time a crop is harmed by one, or when you ask |
| Roles/permissions | More than 10 users, or anyone outside the family |
| Push-notification app instead of SMS | SMS spend above $30/month |
| Hourly rollup aggregate | Year-range chart queries above 1 s |

## 9. Open risks

| Risk | Note |
|---|---|
| Internet outage blind spot | Mitigated only by the cellular alarms. Buy and test them before the first cold night |
| Wi-Fi capacity for ~100 devices per greenhouse | Consumer APs struggle past ~30–50 clients, and humidity and metal frames weaken signal. Plan 2–3 business-grade APs per greenhouse and survey coverage before installing |
| US 10DLC SMS registration | Approval can take days to weeks. Start it now, or use a verified toll-free number |
| OTA reliability across 400 devices | A bad firmware push could silence a greenhouse. Roll out to 5 devices, then one greenhouse, then all |
| Sensor calibration drift | Cheap temperature sensors drift. Compare against a reference thermometer each season |
| Bus factor of one | The runbook plus the independent cellular alarms are the mitigation |
