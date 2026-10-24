# Greenhouse Monitoring: Design

**Summary**
- **Architecture:** one mini PC on site runs an MQTT broker (Mosquitto), a single Python service that stores readings and raises alerts, TimescaleDB, and Grafana. An outside dead-man's switch texts you if the box, the power, or the internet goes dark.
- **Cost:** about $5 to $12/month, plus about $260 of one-time hardware (mini PC and UPS).
- **Top risk:** the box is on site, so an internet outage blocks its texts. The outside watchdog catches that. Wi-Fi coverage for 400 devices is the second big risk.
- **Confirm:** the MQTT topic and payload shape, the sensor ID scheme, the readings schema, the 90-day raw retention, and device timestamps (see Hand-off).

---

## 1. Requirements and numbers

**Confirmed (your defaults):** 3 to 5 greenhouses, consumer broadband, an on-site box. ESP32-class sensors over MQTT. A missed alert is the worst failure, and losing months of history is not OK. 2 to 4 people get texts, with per-greenhouse thresholds, repeats every 15 min until someone acknowledges, and escalation after 15 min. Silence alerts fire at 5 min for a greenhouse and 30 min for a sensor. Retention is raw for 90 days, 1-minute averages for 2 years, and hourly averages forever. Fewer than 10 people use the dashboard, with remote login. Budget is under $50/month and under $300 of hardware. A UPS gets added.

Assumption: each sensor reports one metric (temp, humidity, or soil), so 1 message = 1 row.

| Quantity | Arithmetic | Result |
|---|---|---|
| Writes, average | 400 sensors ÷ 10 s | **40 rows/s** |
| Writes, peak | Sensors flush their buffers after a Wi-Fi drop: 10× | **~400 rows/s** for a minute or so |
| Rows per day | 40 × 86,400 | 3.46M/day |
| Dashboard reads, peak | 10 viewers × 10 panels ÷ 10 s refresh | **~10 queries/s**, usually 0 |
| Alert evaluation | Done in memory on each message | 40 checks/s, negligible |
| Alert fan-out | 1 alert → up to 4 people, plus repeats | ~100 SMS/month in a bad month |
| Raw storage (90 days) | 311M rows × ~75 B ≈ 23 GB uncompressed. Timescale compresses about 10× after 7 days | **~4 GB** steady state |
| 1-min averages | 400 × 1,440 = 576k/day → 210M/yr × ~90 B ≈ 19 GB/yr raw, ~2 GB compressed | **~4 GB** at 2 years |
| Hourly averages | 400 × 24 × 365 = 3.5M/yr | ~300 MB/yr, kept forever |
| Total disk at 2 years | | **~10 GB** (a 500 GB SSD is plenty) |

**Monthly cost**

| Item | $/month |
|---|---|
| Twilio number + ~100 SMS (~$0.01 each incl. carrier fees) | ~$2–3 |
| US A2P 10DLC registration (required for app-sent SMS) | ~$2 (plus ~$20 one-time; check current pricing) |
| Backblaze B2 for backups (~10 GB, plus WAL) | ~$1 |
| Healthchecks.io dead-man's switch | $0 (free tier) to $5 (more SMS/phone-call credits) |
| Cloudflare Tunnel + Access (remote login, <50 users) | $0 |
| Electricity (~10 W box) | ~$1 |
| **Total** | **~$5–12** |

One-time: an N100-class mini PC with 16 GB RAM and a 500 GB SSD (~$170), plus a UPS for the box, router, and main AP (~$90). Use an SSD, not a Raspberry Pi SD card. 3.5M inserts a day wears out SD cards.

**Right-sizing:** 40 writes/s and 10 reads/s are about 1% of what one Postgres server handles. One box, one database, one service. No queue, cache, or cluster.

## 2. Architecture

```
 Greenhouses (Wi-Fi)                  On-site mini PC (on UPS)                    Outside
 ┌──────────────┐  MQTT QoS1  ┌──────────────────────────────────────┐
 │ 400 ESP32    │────────────▶│ Mosquitto (persistent sessions)      │
 │ sensors      │             │        │                             │
 │ (buffer when │             │        ▼                             │
 │  offline)    │             │ greenhouse-svc (Python, systemd)     │  SMS   ┌────────┐
 └──────────────┘             │  ├ ingest: batch insert every 1 s ───┼──────▶ │ Twilio │──▶ phones
                              │  ├ rules: thresholds, silence        │        └────────┘
                              │  ├ notifier: send/repeat/escalate    │  ping  ┌──────────────┐
                              │  └ heartbeat (only if data is fresh) ┼──────▶ │ Healthchecks │──▶ SMS/call
                              │        │                             │  1/min │ (dead-man)   │   if pings stop
                              │        ▼                             │        └──────────────┘
                              │ TimescaleDB (Postgres 16)            │  WAL   ┌──────────────┐
                              │  ├ readings hypertable               ┼──────▶ │ Backblaze B2 │
                              │  ├ 1m / 1h continuous aggregates     │        └──────────────┘
                              │  └ config, alerts, notifications     │ tunnel ┌──────────────┐
                              │ Grafana ◀────────────────────────────┼─────── │ Cloudflare   │◀── family phones
                              │ NUT (UPS monitor → "on battery")     │        │ Tunnel+Access│    (remote)
                              └──────────────────────────────────────┘        └──────────────┘
```

- **Grafana** handles live and historical charts, logins, and the phone view, so you don't build a frontend.
- **The alerter is custom** (about 300 lines). Acknowledge-by-SMS-reply, escalation, and greenhouse-level rules are awkward in Grafana alerting.
- **The dead-man's switch is the key piece.** The box pings Healthchecks every minute, but only if it has processed readings in the last 2 minutes. If the box dies, the power stays out longer than the UPS lasts, the internet drops, or ingest hangs, Healthchecks texts or calls everyone with "greenhouse monitoring offline, check manually."

## 3. Data model

```sql
greenhouses (id smallint PK, name, temp_min_c, temp_max_c, sustain_s int default 120)
sensors     (id text PK,              -- hardware ID burned in firmware, e.g. MAC "a4cf12…"
             greenhouse_id smallint, metric smallint, label, active bool)
contacts    (id, name, phone, escalation_order smallint, active bool)

readings (                            -- Timescale hypertable, 1-day chunks
  time          timestamptz not null, -- device timestamp (NTP-synced)
  sensor_id     text        not null,
  greenhouse_id smallint    not null, -- recorded at write time, so moving a sensor doesn't rewrite history
  metric        smallint    not null, -- 1=temp_c 2=rh_pct 3=soil_pct (new types need no migration)
  value         real        not null,
  UNIQUE (sensor_id, metric, time)    -- makes re-sent readings idempotent
);
-- compress after 7 days, drop raw after 90 days
-- continuous aggregates readings_1m (min/avg/max/count, kept 2 yr) and readings_1h (kept forever)

alerts (id, greenhouse_id, sensor_id null, kind,   -- too_hot | too_cold | gh_silent | sensor_silent | on_battery
        state,                                     -- open | acked | resolved
        opened_at, acked_by, acked_at, resolved_at, escalation_level, next_notify_at);
CREATE UNIQUE INDEX one_open_alert ON alerts (greenhouse_id, kind, coalesce(sensor_id,''))
  WHERE state <> 'resolved';                       -- no duplicate open alerts

notifications (id, alert_id, contact_id, sent_at, twilio_sid, status);
```

**Hot-path query (live greenhouse chart, last hour):**
```sql
SELECT time, sensor_id, value FROM readings
WHERE greenhouse_id = $1 AND metric = 1 AND time > now() - interval '1 hour'
ORDER BY time;
```
**Index:** `(greenhouse_id, metric, time DESC)`. Per-sensor drill-down uses the unique index `(sensor_id, metric, time)`. Charts longer than 6 hours read `readings_1m`, and charts longer than 30 days read `readings_1h`.

## 4. Key flows

**Ingest (hot path)**
1. The sensor publishes to `gh/{greenhouse_id}/{sensor_id}` with QoS 1 and the payload `{"v":1,"t":1729771200,"m":"temp_c","val":24.3}`.
2. The service looks up the sensor (cached in memory), updates `last_seen`, and sends the value to the rule engine.
3. Readings collect in a buffer that is inserted every 1 s, or every 500 rows, with `ON CONFLICT DO NOTHING`.
4. If the device clock is off by more than 5 min from server time, it uses server receive time and counts a `clock_skew` metric.

**Threshold alert**
1. Rule: the **median** of a greenhouse's temperature sensors is outside [min, max] for `sustain_s` (2 min). Using the median means one sensor in direct sun doesn't text you at 3 a.m. A single sensor reading something impossible (below −20 °C or above 60 °C) opens a low-priority `sensor_fault` alert instead.
2. On breach, it inserts the alert row first. The partial unique index blocks duplicates.
3. A notifier loop runs every 30 s and picks up open, unacknowledged alerts whose `next_notify_at` has passed. It texts the level-1 contacts and sets `next_notify_at` to now + 15 min. Each repeat goes one level deeper (level 2, then everyone).
4. **Ack:** someone replies "OK" to the text. The Twilio webhook comes in through the Cloudflare Tunnel, sets the alert to `acked`, and texts the others "Ack'd by Mom." Grafana also links to an ack page as a backup.
5. **Resolve:** the median is back inside the range by 1 °C for 5 min (hysteresis, so it doesn't flap). It sends one "resolved" text.

**Silence:** a loop runs every 30 s. If no sensor in a greenhouse has reported for 5 min, it opens `gh_silent`. A single sensor silent for 30 min opens `sensor_silent`, at low priority, texting only you.

**Power:** NUT sees the UPS switch to battery and opens an `on_battery` alert. A power cut is usually when heaters fail.

**Weekly self-test:** every Monday at 9:00 it sends a test text to everyone. That proves the whole SMS path works, including Twilio, account balance, and 10DLC status.

**Background (Timescale jobs):** compression, retention, and aggregate refresh. Their failures show up in `timescaledb_information.job_errors`, which the service checks hourly and alerts you about.

## 5. Data protection

| Data | Cost to lose it | RPO | Mechanism |
|---|---|---|---|
| Config (thresholds, contacts, sensor map) | High. Wrong thresholds mean no alerts | ~5 min | In Postgres, plus WAL archiving to B2 (pgBackRest). Also export to a JSON file in git after each change |
| Alert and ack history | Medium | ~5 min | Same WAL archive |
| Raw readings (90 days) | Low per minute, high per month | ~5 min | WAL archive with `archive_timeout = 300`, plus a weekly full backup |
| 1-min and hourly averages | High after 90 days. They are the only copy once raw data is dropped | ~5 min | Same, so point-in-time recovery covers all of it |
| Readings during a box outage | Low | Sensor buffer size | ESP32 keeps about 10 min of readings in RAM. Longer gaps are lost, and that's accepted |

- **Availability target:** 99.5% for the alerting path (~3.6 h/month), since there is one box. What matters more is detecting downtime within 5 min, and the dead-man's switch does that.
- **RTO:** 2 hours. Restore from B2 onto a spare laptop or a new mini PC, following a written runbook. Practice the restore once before the first cold season, then once a year.

## 6. Decisions

| Choice | Alternative | Why | Hard to reverse? |
|---|---|---|---|
| MQTT topic `gh/{gh}/{sensor}` + versioned JSON payload (`"v":1`) | HTTP POST per reading | Built for flaky Wi-Fi and cheap on the ESP32. The version field allows future changes | **Yes.** Lives in firmware on 400 devices |
| OTA firmware updates from day one | Flash by USB | Without it, every firmware change means walking to 400 sensors | **Yes.** Can't add OTA later without one USB pass |
| Sensor ID = hardware ID, mapped to a greenhouse in the DB, with `greenhouse_id` stored on each reading | Location-based IDs ("GH2-T07") | Moving or replacing a sensor is a DB edit, and history stays correct | **Yes.** Rewriting IDs means migrating all history |
| Narrow readings schema (one row per metric) | Wide row (temp, rh, soil columns) | New sensor types (CO₂, light) need no schema change. Compression erases the size difference | **Yes.** Migrating hundreds of millions of rows |
| Device timestamp as reading time | Server receive time | Buffered readings land at the right time after a Wi-Fi drop | **Yes.** Mixing the two later leaves history inconsistent |
| Retention: raw 90 days, 1-min 2 years, hourly forever | Keep raw forever (~8 GB/yr compressed, also affordable) | Enough to diagnose a recent incident | **Yes, one way.** Dropped raw data can't come back |
| TimescaleDB | Plain Postgres; InfluxDB | Still Postgres for alerts and config, and adds compression, retention, and aggregates | Somewhat. Data migration, but pg_dump-friendly |
| On-site box | Cloud VM | Data keeps flowing and the LAN dashboard works during internet outages. The dead-man's switch covers what an on-site box can't | No. Dump, restore, and change the broker hostname (use a DNS name in firmware) |
| Grafana for charts | Custom React app | Weeks of frontend work saved | No |
| Custom alerter in the same process | Grafana alerting | Ack-by-reply, escalation, median rules | No |
| Twilio | Other SMS APIs | Inbound replies, reliable. Start 10DLC registration now, since approval can take weeks | No |
| Median-of-greenhouse rule, 2 min sustain | Any single sensor | Fewer false 3 a.m. texts. Single-sensor faults are tracked separately | No (config) |

## 7. Failure walk

| Scenario | What happens |
|---|---|
| **Service crashes mid-batch** | Up to 1 s of readings is lost (accepted). systemd restarts it in about 5 s. Mosquitto's persistent session queues messages while it's down. In-memory sustain timers reset, which can delay an alert by up to 2 min |
| **Crash between alert insert and SMS** | The alert row exists with `next_notify_at` in the past, so the notifier sends on restart |
| **Crash after Twilio sent, before recording** | One duplicate text. A duplicate is better than none |
| **Sensors re-send after reconnecting** | The unique key with `ON CONFLICT DO NOTHING` drops the duplicates |
| **Duplicate alert / double ack / Twilio webhook retry** | The partial unique index allows one open alert. Ack is an idempotent state update keyed on the Twilio `MessageSid` |
| **Twilio down or slow** | 10 s timeout and 3 retries with backoff. Then it calls Healthchecks' `/fail` on a separate "alert-delivery" check, so Healthchecks texts or calls everyone through its own provider |
| **Internet down** | The box can't text. Heartbeats stop, and Healthchecks alerts everyone within 5 min: "monitoring offline." Data keeps recording locally and WAL archives queue. The LAN dashboard still works |
| **Power cut** | The UPS keeps the box, router, and main AP up for about 1 to 2 h, and an `on_battery` alert fires. After that, the dead-man's switch fires |
| **A greenhouse AP dies** | `gh_silent` alert within 5 min |
| **Box hardware dies** | Dead-man's switch alerts within 5 min. Restore from B2 with the runbook (RTO 2 h) |
| **B2 down** | WAL queues on local disk. The service alerts you if archive lag exceeds 1 h (`pg_stat_archiver`) |
| **Cloudflare down** | Remote dashboard and SMS-reply acks fail. Ack from the LAN page, or the alert keeps repeating, which is safe |
| **Non-code SPOFs** | Domain (used for the tunnel): auto-renew, with a calendar reminder 30 days before expiry. TLS: Cloudflare handles it. Twilio balance: auto-recharge, and the weekly test catches failures. 10DLC campaign suspended: weekly test. Contact phone numbers change: the weekly test texts everyone, so stale numbers show up. Timescale jobs silently stop: hourly `job_errors` check. Disk full: alert at 80%. Clock drift: NTP on the box plus the skew metric. The ingest process hangs but stays "up": the heartbeat requires fresh data, not just a live process |

## 8. Not building yet

| Item | Add it when |
|---|---|
| Cellular SMS modem on the box (~$50 + ~$5/mo SIM) | More than 1 internet outage over 15 min per month, or any outage on a frost-risk night |
| Second box or cloud failover for alerting | Unplanned box downtime over 1 h happens twice in 6 months |
| Cloud copy of data and dashboard | More than 10 remote viewers, or a need to see charts during internet outages |
| Message queue (Kafka etc.) | More than 5,000 readings/s sustained (~50,000 sensors at 10 s) |
| Separate alerter process | Ingest code is deployed more than weekly, or restarts delay an alert by more than 2 min more than once |
| Custom web app | Family edits thresholds more than monthly and Grafana's UI is the blocker |
| Rate-of-change / predictive alerts | After one season, if a threshold alert ever came too late to save plants |
| Caching or read replicas | Dashboard over 500 queries/s (about 50× today) |

## 9. Open risks

- **Wi-Fi capacity and coverage.** 400 clients on consumer APs is a lot. Plan for about 50 to 80 sensors per AP, so 5 to 8 APs on 2.4 GHz. Metal frames and humidity hurt range. Run a pilot of 20 sensors in one greenhouse first.
- **A single on-site box.** You will always *know* when it's down, but nothing fixes it automatically. Family members need a one-page "if you get the 'monitoring offline' text" runbook.
- **SMS deliverability.** 10DLC registration can take weeks. Don't go live without it, because unregistered texts get filtered.
- **Sensor placement and calibration.** Direct sun or a heater vent reads wrong. Do a 2-week burn-in before trusting thresholds.
- **Alert fatigue.** If false alarms happen weekly, people stop reading. Review every alert after the first month and tune the sustain and hysteresis settings.
- **Bus factor of one.** Keep the system in git with the runbook and a `make restore` script.

---

## Hand-off: please confirm before any code is written

**The 5 most consequential hard-to-reverse decisions**

1. **MQTT topic and payload shape** (`gh/{gh}/{sensor}`, JSON with `"v":1`). Changing it later means reflashing all 400 sensors, or supporting both formats forever.
2. **OTA firmware updates from day one.** Skipping it means a USB walk to 400 devices for every firmware fix.
3. **Sensor ID = hardware ID, with `greenhouse_id` stored on each reading.** Changing it later means rewriting hundreds of millions of history rows.
4. **Narrow readings schema** (one row per metric). Switching to wide later means a full table migration and rebuilding the aggregates.
5. **Retention: raw 90 days, then 1-min for 2 years, hourly forever.** Raw data that has been dropped can't be recovered. Keeping raw forever costs only about 8 GB/yr, so say so if you'd rather.

**Other hard-to-reverse decisions**
- Device timestamps (NTP) as reading time, with server time as a fallback.
- TimescaleDB as the store (migration possible, but it's real work).

Reply "confirmed" to accept all of these, or tell me what to change.
