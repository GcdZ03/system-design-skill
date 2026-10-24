**Summary**
- **Architecture:** sensors publish over MQTT/TLS to one cloud VM (Mosquitto, Postgres + TimescaleDB, and an app split into three processes: ingest, alerter, web), with SMS through Twilio and an external dead-man's switch watching the alerter.
- **Cost:** about $35/month, plus about $150–300 one-time per site for an independent cellular temperature alarm.
- **Top risk:** if a site's internet drops, the cloud can't see the heat. You get a "site offline" text within 5 minutes, but no temperature alert. The cellular alarm covers that gap.
- **Confirm:** the MQTT topic/payload contract flashed into the firmware, hardware-MAC sensor IDs, 90-day raw retention, and storing each reading's greenhouse at write time.

The full design is saved at `<repo>/evals/runs/2026-10-24-draft3/sensors/DESIGN.md`.

**Most consequential hard-to-reverse decisions**
1. **MQTT topic + versioned JSON payload in firmware** (`gh/{site}/{greenhouse}/{mac}`, `{"v":1,...}`). Changing it later means an OTA update to all 400 devices, and any device without working OTA has to be reflashed by hand.
2. **Raw 10-second readings kept 90 days, 1-minute rollups kept forever.** Once raw data is dropped it's gone, so you can't later decide you wanted last year's 10-second detail.
3. **Each reading stores its greenhouse at write time.** Adding this later means a backfill over 1B+ rows, with guesses about where sensors were in the past.
4. **Sensor ID = the ESP32's hardware MAC.** It's in every reading row and every device credential, so switching ID schemes means a full data migration plus new device credentials.
5. **TimescaleDB for storage.** Moving off it later means migrating every hypertable and rebuilding compression, retention, and rollups.

**Other hard-to-reverse decisions**
- Store values in SI units (°C, %) and convert to °F only in the UI.
- Use your own long-lived CA, flashed into firmware, for device TLS instead of Let's Encrypt.

Please confirm all of these in one reply, or say what to change, before any code is written.
