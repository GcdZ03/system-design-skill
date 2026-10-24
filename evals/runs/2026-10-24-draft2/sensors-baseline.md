# Greenhouse Sensor Monitoring: System Design

## 1. What the numbers say

- 400 sensors x 1 reading / 10 s = **40 messages/second**. Each reading is about 50 bytes, so roughly 2 KB/s.
- Per day that's about **3.5 million rows**. Per year it's about **1.26 billion rows**, or 15-25 GB raw in a time-series database before compression, and far less after.

This is a small workload. A single modest machine handles it with a lot of room to spare. You don't need Kafka, Kubernetes, microservices, or a distributed database. Scale isn't the hard part here.

**Alerting reliability is.** The failure that kills a crop at 3 a.m. usually isn't "the temperature went out of range and we noticed." It's that the heater failed *because the power went out*, and the power outage also took down the Wi-Fi, the sensors, and your server, so nothing alerted at all. Most of this design is about making sure silence never gets read as "everything is fine."

## 2. Architecture

```
 [Sensors: ESP32 + SHT40 / soil probe]  x400
        |  Wi-Fi, MQTT over TLS, QoS 1
        v
 [MQTT broker: Mosquitto]  -------------------------------+
        |                                                 |
        v                                                 |
 [Ingest service]  -> batch insert ->  [TimescaleDB (Postgres)]
        |                                   |
        v                                   v
 [Alert evaluator] --> Twilio SMS      [Grafana: live + historical dashboards]
        |
        +--> heartbeat ping --> [External dead-man's switch, e.g. Healthchecks.io] --> SMS/call

 Independent backup: one cellular, battery-backed temperature alarm per greenhouse
```

### Components

**Sensors (firmware)**
- ESP32-class boards are cheap, have Wi-Fi, and have good library support (ESPHome or Arduino + PubSubClient).
- Publish to a topic like `gh/{greenhouse_id}/{sensor_id}/reading` with a JSON payload: `{ts, temp_c, rh, soil, battery_v, rssi, fw}`.
- Include `rssi` and `battery_v`. They let you tell a dying sensor apart from a real reading problem.
- Set an MQTT **Last Will and Testament** so the broker publishes `offline` when a sensor drops.
- Keep a small local ring buffer and replay it on reconnect (with original timestamps) so short Wi-Fi drops don't leave gaps in the history. Get time from NTP.
- If you can, use ESPHome. It gives you OTA updates, Wi-Fi reconnect, and MQTT for free, and that's a big deal for one developer managing 400 devices.

**MQTT broker: Mosquitto**
- One process that easily handles thousands of clients. Use TLS and per-device credentials (or at least a per-greenhouse credential) so a stolen device can't be used to spoof the others.
- MQTT beats HTTP POST here: persistent connections, low overhead on the microcontroller, QoS 1 delivery, and LWT for offline detection.

**Ingest service**
- A small Python or Go process subscribes to `gh/#`, validates payloads, drops physically impossible values (e.g. -40 C or 120 C from a sensor glitch), and batch-inserts every 1-2 seconds.
- You could use Telegraf instead of writing this, but you'll want custom validation and the alert hook, so a ~200-line service is reasonable.

**Storage: TimescaleDB (Postgres extension)**
- Why: it's just Postgres, so you also get normal tables for greenhouses, sensors, thresholds, contacts, and alert history in one database you already know how to back up. InfluxDB is a fine alternative if you prefer it.
- Schema:
  ```sql
  CREATE TABLE readings (
    ts          timestamptz NOT NULL,
    sensor_id   int         NOT NULL,
    temp_c      real,
    rh          real,
    soil        real,
    battery_v   real,
    rssi        smallint
  );
  SELECT create_hypertable('readings', 'ts');

  CREATE TABLE sensors (
    sensor_id     serial PRIMARY KEY,
    greenhouse_id int  NOT NULL,
    zone          text,
    kind          text,          -- air / soil
    active        boolean DEFAULT true
  );
  ```
- **Continuous aggregates**: 1-minute and 1-hour rollups (min/avg/max per sensor and per greenhouse). Historical charts query these, never the raw table, so a 90-day chart stays fast.
- **Retention**: raw 10-second data for 30-90 days, 1-minute data for 2 years, 1-hour data forever. Turn on compression for chunks older than 7 days. The whole thing then fits in a few GB.

**Dashboard: Grafana**
- Don't build a custom frontend. Grafana connects straight to TimescaleDB and gives you live panels (5-10 s refresh), historical zooming, per-greenhouse views, and mobile-friendly access with logins.
- Suggested dashboards: a farm overview (one tile per greenhouse showing its current median temperature, colored by status), a greenhouse detail view (every sensor in that house), and sensor health (last-seen time, battery, RSSI).
- If you later want a polished custom app, it can read from the same database. Start with Grafana.

**Alert evaluator**
You could use Grafana alerting, but I'd write a small dedicated service (or make it part of the ingest process), because this is the part that has to be right, and you want to test it on its own. Rules:

1. **Alert on the greenhouse, not the single sensor.** Use the median (or the second-highest/second-lowest) of the air sensors in a house. One sensor sitting in direct sun or with a bad probe shouldn't wake anyone up, and one dead sensor shouldn't hide a real problem.
2. **Require the condition to persist.** For example, over the limit for 3 consecutive minutes. Use hysteresis so it doesn't flap: alert at 35 C, clear only below 33 C.
3. **Two severities.** A warning (approaching the limit) by SMS to one person. A critical alert (at the limit, or the temperature changing fast, e.g. dropping more than 3 C in 15 minutes at night) by SMS plus a phone call to everyone on the rota.
4. **Staleness alerts.** "No data from Greenhouse 3 for 5 minutes" is just as critical as "too cold." Track last-seen per greenhouse and per sensor. A whole house going quiet is critical, one sensor going quiet is a daytime maintenance notice.
5. **Escalate and acknowledge.** Repeat every 10-15 minutes until someone replies "ACK" by SMS (Twilio inbound webhook), then escalate to the next person. A single text that arrives while someone's asleep isn't enough.
6. **Low-priority notices** (sensor battery low, weak signal, soil moisture out of range) go into a daily digest, not a 3 a.m. text.
7. Save every alert and acknowledgment in an `alerts` table so you can review what happened afterward.

Thresholds belong in a database table per greenhouse (crops differ), editable from a simple admin page or even straight from Grafana/psql at first. Don't hardcode them.

**SMS delivery**: Twilio (or a similar provider) for SMS and voice calls. Budget a few dollars a month. Configure at least two recipients.

## 3. Making silence loud (the most important section)

Every layer can fail silently, so each one needs a watcher outside itself:

| Failure | How you find out |
|---|---|
| One sensor dies | Per-sensor last-seen check, daily digest |
| Greenhouse Wi-Fi/AP dies | Greenhouse-level staleness alert (critical) |
| Ingest or alert service crashes | It pings an **external dead-man's switch** (Healthchecks.io, Cronitor, or Better Uptime) every minute. If the pings stop, *that* service texts or calls you. |
| Server or site internet dies | Same dead-man's switch, because it runs off-site |
| Site power outage | The dead-man's switch fires. In addition, put the AP, router, and server on a **UPS**, and add a **cellular backup** (LTE router failover) |
| Everything above fails at once | An **independent cellular temperature alarm** in each greenhouse (off-the-shelf, battery-backed, about $100-250 each) |

That last row is deliberate. For "crop dies overnight" stakes, a cheap standalone device that texts on its own cellular connection is the right backstop for software you build and run alone. It has nothing in common with your system, so it doesn't fail when your system does. I'd install those *first*, before writing any code.

Also run the services under systemd (or Docker with `restart: always`) so they come back automatically after a crash or reboot.

## 4. Where to run it

Two reasonable options:

**A. On-site mini PC (e.g. Intel NUC, ~$300) on a UPS**: what I'd recommend.
- Sensor traffic never leaves the LAN, so internet outages don't affect data collection, and history keeps recording.
- Alerts need outbound internet. Cover that with the LTE failover and the external dead-man's switch.
- For remote dashboard access, use Tailscale or Cloudflare Tunnel instead of opening ports.

**B. Small cloud VM (~$10-20/month)**
- Easier to reach from anywhere, and no hardware to babysit.
- But every site internet outage becomes a data gap *and* a blind spot, and 400 devices keep persistent connections over the WAN.

A hybrid also works if there are several sites: a Mosquitto broker on each site bridges to a central cloud broker, which buffers data during outages. With a few greenhouses that may share a site, option A is simpler.

**Backups**: nightly `pg_dump` (or pgBackRest) to off-site object storage (Backblaze B2 / S3). Test a restore once.

## 5. Wi-Fi reality check

400 Wi-Fi clients across greenhouses is mostly a network problem, not a software one:
- Use commercial access points (UniFi, Omada, Aruba Instant On). Consumer routers start failing at around 30-50 clients.
- Greenhouses are humid and full of water-filled plants and metal framing, so do a site survey and put APs inside each house in weatherproof enclosures.
- Put sensors on their own 2.4 GHz SSID and VLAN.
- Ten-second reporting with Wi-Fi always on rules out batteries. Plan for wired USB power, or accept a longer interval with deep sleep (60 s or more is plenty for greenhouse air temperature). Honestly, 10 seconds is more frequent than greenhouse thermal behavior needs. 30-60 s would cut load, power, and storage by 3-6x and lose almost nothing. Consider it.
- If Wi-Fi coverage turns out to be painful, LoRaWAN sensors with one gateway per site are the usual alternative for agriculture.

## 6. Build order for one developer

1. **Week 0:** Install the standalone cellular temperature alarms. The crop is now protected while you build.
2. **Week 1:** Ten sensors in one greenhouse -> Mosquitto -> ingest -> TimescaleDB -> Grafana. Get a chart on screen.
3. **Week 2:** Alert evaluator with Twilio, staleness checks, the external dead-man's switch, and escalation/ACK. Test by unplugging things: a sensor, the AP, the server, the internet.
4. **Week 3:** Continuous aggregates, retention, compression, backups, UPS, LTE failover.
5. **Then:** Roll out to all 400 sensors one greenhouse at a time, watching sensor-health panels (RSSI, drop rate) as you go.

## 7. Rough costs

- Server: ~$300 one-time (or ~$15/month cloud)
- UPS + LTE failover router: ~$300-500 one-time + ~$10-20/month data
- Sensors: ~$10-25 each for ESP32 + SHT40 + capacitive soil probe + enclosure, or ~$4k-10k for 400
- Commercial APs: ~$100-200 each, a few per greenhouse
- Twilio: a few dollars a month
- Healthchecks.io: free tier is enough
- Standalone cellular alarms: ~$100-250 per greenhouse + small monthly fee

## 8. Questions that would change the design

- Are the greenhouses at one site or several? (Several pushes toward the hybrid broker setup.)
- Do sensors have mains power, or do they need to run on batteries?
- Do you want to *control* anything later (vents, heaters, irrigation)? If so, keep control loops on local controllers, never dependent on this server, and treat this system strictly as monitoring.
- Who is on call at night, and do they reliably get texts? (That decides your escalation chain.)

**Summary:** ESP32 sensors -> MQTT -> one small server running Mosquitto, an ingest/alert service, TimescaleDB, and Grafana, with Twilio for SMS and calls. Every layer is watched by something outside it, and an independent cellular alarm in each greenhouse is the final backstop. The data volume is easy. Spend your effort on alerts that still fire when the power goes out.
