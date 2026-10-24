Before I design anything, a few questions. Each one could change the architecture or the cost. My proposed default is in brackets.

For scale, 400 sensors every 10 seconds is 40 writes/s, about 3.5M readings a day and about 1.26B a year. That's why retention is question 1.

1. **Retention.** How long do you need the raw 10-second readings, and how long do you need coarser history? [Raw for 90 days. 1-minute averages kept forever for season-over-season comparison.]
2. **Downtime vs. data loss.** Which is worse? [Missing a hot/cold alert is by far the worst. Losing a few minutes of chart data is fine. A dashboard outage of an hour is fine.]
3. **Budget and operations.** What's the monthly budget, and is it only you running it, including at 2 a.m.? [Under $50/month. You alone, no on-call rotation.]
4. **Internet at the greenhouses.** How reliable is it, and is there any backup link? If the internet drops, a cloud-only system can't see the heat, so it can't send the alert. [One broadband line per site, no backup. It goes down a few times a year.]
5. **Sensor hardware.** What are the sensors, and can you control what they send and where? [ESP32-class boards you flash yourself, able to send MQTT or HTTP to any address. Not a vendor's closed cloud.]
6. **Sites and greenhouses.** How many greenhouses, at how many physical sites? [4 greenhouses across 2 sites, about 100 sensors each.]
7. **Alert rule.** What counts as "too hot" or "too cold"? [Per-greenhouse min/max on the median temperature over the last 5 minutes. This stops one faulty sensor from paging you. Also alert if a whole greenhouse goes silent for 5 minutes.]
8. **Who gets texted, and should it escalate?** [Up to 4 family phone numbers. Text everyone, then repeat every 15 minutes until someone replies "ACK" or the temperature recovers.]
9. **Dashboard users.** Who looks at it and how? [Up to 5 family members, mostly on phones, all with the same permissions, each with their own login. Live charts refresh every 10–30 s.]

**Assumptions** (tell me if any are wrong):
- Humidity and soil moisture are only charted, with no alerts on them for now.
- The system only monitors. It doesn't control heaters, vents, or irrigation.
- No compliance requirements (no food-safety audit trail).
- SMS through a provider like Twilio, costing a few cents per text.
- Sensors get their clocks from NTP, and the server timestamps readings that arrive with bad clocks.
- The dashboard is reachable over the internet, not only on the farm network.
- Alert thresholds can change by crop and season, and you edit them in the dashboard.

Reply with changes, or "defaults" to accept all.
