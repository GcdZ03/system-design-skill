Before I design anything, a few questions. Each answer could change the architecture or the cost, so I've put my proposed default in brackets. You already told me the sensor count (about 400), the reporting interval (10 s), the transport (Wi-Fi), what you need (live and historical charts plus text alerts), and who builds it (you). I won't ask about those again.

1. **Sites and internet.** How many greenhouses are there, are they all on one property, and how reliable is the internet there? [3 to 5 greenhouses on one or two properties, consumer broadband that drops now and then, no cellular backup]
2. **Where it runs.** Cloud VM, or a small box on site such as a mini PC or Raspberry Pi? This is the biggest fork in the design, because alerts have to keep working when the internet is down. [A small box on site for collecting data and alerting, plus an optional cloud copy for viewing the dashboard away from the farm]
3. **Sensor hardware and protocol.** What are the sensors, and what can they send: MQTT, HTTP POST, or vendor cloud only? [ESP32-class boards you can flash, publishing MQTT]
4. **Which is worse.** Missing an alert, or losing historical readings? [Missing an alert is far worse. Losing a few minutes of history is acceptable. Losing months of history is not.]
5. **Alert rules.** Who gets texted, and how should alerts behave? Do thresholds differ per greenhouse or crop? Do you need acknowledgement and escalation, e.g. if nobody acknowledges within 10 minutes, text the next person? [2 to 4 family members, thresholds set per greenhouse, repeat every 15 min until someone acknowledges, escalate to a second person after 15 min]
6. **Silent failures.** Should a sensor or greenhouse that stops reporting trigger an alert too? A dead sensor during a cold snap looks just like a quiet night. [Yes. Alert if a greenhouse has no readings for 5 minutes, or if a single sensor has been silent for 30 minutes]
7. **History retention and resolution.** How long do you keep data, and at what detail? [Raw 10 s readings for 90 days, 1 minute averages for 2 years, hourly averages forever]
8. **Dashboard users.** How many people look at it, and from where? [Fewer than 10 people, phone and laptop, sometimes from off the farm, so it needs a login]
9. **Budget.** How much per month, including SMS? [Under $50/month, plus a one-time hardware spend under $300]
10. **Power.** Is there backup power for the Wi-Fi and the server? A power cut is also when heaters fail. [No UPS today. I'll suggest adding a small UPS and treat that as something you'd buy.]

**Assumptions** (tell me if any are wrong):
- Readings are small: sensor ID, timestamp, and 1 to 3 values. No images or video.
- "Too hot" and "too cold" are simple threshold rules, possibly with a short sustain window (e.g. above 35 °C for 2 minutes). No prediction or ML.
- No compliance requirements such as food-safety audit logs.
- The dashboard is read-only for most users, and only you change thresholds and sensor config.
- Sensors don't need two-way control, such as turning heaters on. This is monitoring only.
- You're comfortable with Linux, Docker, and one mainstream language (Python or Node).
- An SMS provider such as Twilio is acceptable for sending texts.

Reply with changes, or 'defaults' to accept all.
