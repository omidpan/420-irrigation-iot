# Irrigation IoT: store-and-forward from Raspberry Pi to Chart.js and Grafana

Three dummy sensors for irrigation (temperature, humidity, water pressure) on a
Raspberry Pi 5. Every reading is saved in a local SQLite buffer on the Pi, then
forwarded over MQTT. If the laptop or the network is down, readings wait on the Pi
and are sent later with their original times. The laptop runs everything else in
Docker Compose.

```
Raspberry Pi 5 (Python venv)                         Laptop (Docker Compose)
┌───────────────────────────────┐                    ┌────────────────────────────────────────┐
│ sensors.py  (dummy values)    │                    │ mosquitto   :1883                      │
│      │                        │  MQTT QoS 1        │     │                                  │
│      ▼                        │  irrigation/       │     ▼                                  │
│ pi_buffer.db  (outbox)  ──────┼──<device>/readings─┼► subscriber ─► data/irrigation.db      │
│  pending until PUBACK         │                    │                  │            │        │
│ node.py: OFFLINE after 60 s,  │                    │                  ▼            ▼        │
│ forwards backlog oldest-first │                    │   web (Flask + Chart.js)   grafana     │
└───────────────────────────────┘                    │        :5000                :3000      │
                                                     └────────────────────────────────────────┘
```

## Folder layout

```
irrigation-iot/
├── pi/                          runs on the Raspberry Pi (virtual environment)
│   ├── node.py                  main loop: read → buffer → forward
│   ├── db.py                    SQLite outbox (pi_buffer.db)
│   ├── sensors.py               dummy temperature / humidity / water pressure
│   ├── requirements.txt         paho-mqtt 2.x
│   ├── irrigation-node.service  optional: start on boot with systemd
│   └── Dockerfile               only for the simulated Pi on the laptop
└── local-server/                       runs on the laptop
    ├── docker-compose.yml
    ├── mosquitto/config/mosquitto.conf
    ├── common/                  db.py (schema) + dummy_sensors.py, copied into images
    ├── subscriber/              MQTT → validation → SQLite, plus seed_history.py
    ├── web/                     Flask API + Chart.js page
    ├── grafana/                 provisioned data source + dashboard
    └── data/                    irrigation.db appears here
```

## 1. Laptop: start the stack

```bash
cd local-server
docker compose up -d --build
docker compose ps                 # mosquitto, subscriber, web, grafana: running
```

- Flask + Chart.js: http://localhost:5000
- Grafana: http://localhost:3000 (anonymous view; admin login `admin` / `iot420`), dashboard **IoT → Irrigation Monitor**

The first Grafana start needs internet to install the SQLite plugin.

### Dummy history right away (optional)

```bash
docker compose run --rm subscriber python seed_history.py --hours 24
```

This writes 24 hours of readings (one per minute) as device `pi5-demo`. To remove them later:
`DELETE FROM sensor_readings WHERE device = 'pi5-demo';`

## 2. Raspberry Pi: run the node

Copy the `pi/` folder to the Pi, then:

```bash
cd ~/irrigation-iot/pi
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt

# use your laptop's IP address (ipconfig / ip a)
MQTT_HOST=192.168.1.50 DEVICE_ID=pi5-01 python node.py
```

Only `paho-mqtt` is installed. `sqlite3`, `json`, `uuid`, `contextlib` and the rest are
in Python's standard library. Check the buffer at any time with `python db.py`.

Settings are environment variables (defaults in brackets):

| Variable                                        | Meaning                                                             |
| ----------------------------------------------- | ------------------------------------------------------------------- |
| `MQTT_HOST` [192.168.1.50] / `MQTT_PORT` [1883] | the laptop running Mosquitto                                        |
| `DEVICE_ID` [pi5-01]                            | lowercase letters, digits, hyphens                                  |
| `SAMPLE_SECONDS` [5]                            | time between readings                                               |
| `OFFLINE_AFTER_SECONDS` [60]                    | no delivery for this long = OFFLINE, keep buffering                 |
| `PUBLISH_TIMEOUT_SECONDS` [5]                   | wait this long for the broker's PUBACK                              |
| `KEEP_SENT_DAYS` [7]                            | delivered rows kept in pi_buffer.db; pending rows are never deleted |

No Pi at hand? Simulate one on the laptop:
`docker compose --profile simulator up -d --build` (device `pi5-sim`).

## 3. Test the store-and-forward workflow

1. With everything running, watch the Pi log: `Read …` every 5 s, no warnings.
2. Stop the broker on the laptop: `docker compose stop mosquitto`.
3. After about 60 s the Pi logs `OFFLINE for 60 s: buffering readings on the Pi (n pending)`.
   `python db.py` on the Pi shows the pending count growing.
4. Wait 2 to 3 minutes, then `docker compose start mosquitto`.
5. The Pi logs `ONLINE again: forwarding backlog` and the pending count drops to 0.
6. Check the result:
   - Grafana **Delivery delay** panel: a ramp of points up to ~180 s (the buffered readings), then back to 0.
   - Grafana **Buffered readings (last hour)** and the Flask "Store-and-forward" card show how many arrived late.
   - The history charts have **no gap**: every reading kept its original collection time.
   - The Flask table marks late rows as `buffered, +Ns`.

Other tests:

- Stop the subscriber instead (`docker compose stop subscriber`). Mosquitto keeps the QoS 1 messages for it (persistent session) and delivers them on start.
- Pull the Pi's network cable or turn off Wi-Fi: same result as step 2.
- Duplicates: a message sent twice (after a connection drop before the PUBACK) is ignored thanks to the unique `msg_id`.

## How each part works

**Pi (`node.py`)** writes every reading to `outbox` first, then sends pending rows oldest-first
with QoS 1, and marks a row as sent only after `wait_for_publish()` confirms the broker's PUBACK.
Writing first means nothing is lost even if power fails mid-send. A Last Will message
(`irrigation/<device>/status = offline`, retained) tells the laptop when the Pi disappears.

**Subscriber** validates each message (JSON, device name, UTC timestamp not in the future,
numeric ranges, at least one value) and runs `INSERT OR IGNORE` on the unique `msg_id`.
It stores two times: `collected_at` (on the Pi) and `received_at` (on the laptop), both UTC.
The difference is the delivery delay.

**Flask** serves `/api/devices`, `/api/latest`, `/api/history?minutes=15|60|1440` (averaged into
~300 points) and `/api/recent`, each with an optional `&device=` filter. `static/app.js` creates
the Chart.js charts once and updates them every 5 s.

**Grafana** reads the same `data/irrigation.db` through the provisioned SQLite data source.
Time series queries convert `collected_at` to epoch seconds and filter with `$__from` / `$__to`.

## Add a new device (another Pi)

Nothing changes on the laptop. The subscriber listens to `irrigation/+/readings`, so
any device name is accepted, and both dashboards pick it up automatically.

**A real Raspberry Pi:** copy the `pi/` folder to it, set up the virtual environment
(section 2) and start it with its **own** `DEVICE_ID`:

```bash
MQTT_HOST=<LAPTOP_IP> DEVICE_ID=pi5-02 python node.py
```

`DEVICE_ID` rules: lowercase letters, digits and hyphens, at most 32 characters, unique per device.
Two Pis with the same ID would look like one device.

**A second simulated Pi:** add a service to `docker-compose.yml` (next to `pi-sim`) and a volume
for its buffer:

```yaml
  pi-sim-2:
    profiles: ["simulator"]
    build: ../pi
    container_name: pi-sim-2
    environment:
      DEVICE_ID: pi5-sim2
      MQTT_HOST: mosquitto            # use the same value as pi-sim
      PI_DB_PATH: /buffer/pi_buffer.db
    volumes:
      - pi-sim-2-buffer:/buffer
    depends_on:
      - mosquitto
    restart: unless-stopped

volumes:
  # ... keep the existing ones and add:
  pi-sim-2-buffer:
```

```powershell
docker compose --profile simulator up -d --build
```

**Chart.js:** the **Device** dropdown in the header lists every device found in the database
(new ones appear within 5 seconds). Choose one to see only its data. **All devices** averages
them together. The API takes the same filter: `/api/history?minutes=60&device=pi5-02`,
`/api/latest?device=pi5-02`, `/api/recent?device=pi5-02`, and `/api/devices` lists them all.

**Grafana:** the provisioned panels show all devices together. To choose devices:

1. Dashboard → **Edit** → **Settings → Variables → New variable**.
   - Type **Query**, name `device`, data source `Irrigation (SQLite)`, query
     `SELECT DISTINCT device FROM sensor_readings ORDER BY device`.
   - Turn on **Multi-value** and **Include All option**. Save.
   - If the SQLite data source is not offered for Query variables in your Grafana version,
     use type **Custom** instead and type the names: `pi5-01,pi5-02,pi5-sim`.
2. In each panel query, add this line to the `WHERE` part (for the Stat panels, add a `WHERE` before `ORDER BY`):

   ```sql
   AND device IN (${device:singlequote})
   ```

3. One line per device in a Time series panel: add `device` to the `SELECT`, for example
   `SELECT CAST(strftime('%s', collected_at) AS INTEGER) AS time, device, temperature AS "Temperature"`,
   then **Transformations → Add → Partition by values** → field `device`.
4. Save, then **Export → Export as JSON** and overwrite `grafana/dashboards/irrigation.json`
   so the change survives a rebuild.

## Add a new sensor (example: soil moisture, %)

A new measurement travels through every layer, so it is added in every layer. Use the
same name everywhere: `soil_moisture`. Lower-case with underscores keeps SQL, Python and
JavaScript happy.

**1. Pi: produce the value** in `pi/sensors.py`, inside `read()`:

```python
soil_moisture = 35 - 10 * day_wave + self.rng.gauss(0, 1)   # dummy %, wetter at night
return {
    "temperature": round(temperature, 1),
    "humidity": None if humidity is None else round(humidity, 1),
    "water_pressure": round(water_pressure, 1),
    "soil_moisture": round(soil_moisture, 1),               # NEW
}
```

Copy the same file to `local/common/dummy_sensors.py` (used by `seed_history.py`).
Nothing else changes on the Pi: `node.py` sends whatever `read()` returns.

**2. Database: add the column.** In `local/common/db.py`, add it to `SCHEMA` for new databases:

```sql
    water_pressure REAL,                   -- kPa   (add the comma)
    soil_moisture  REAL                    -- %     NEW
```

`CREATE TABLE IF NOT EXISTS` does not change a table that already exists, so add the column
to your current database once:

```powershell
docker compose exec subscriber python -c "import sqlite3; c=sqlite3.connect('/data/irrigation.db'); c.execute('ALTER TABLE sensor_readings ADD COLUMN soil_moisture REAL'); c.commit(); print('column added')"
```

**3. Subscriber: accept and store it.**

`local/subscriber/validation.py`, add the allowed range:

```python
RANGES = {
    "temperature": (-40.0, 85.0),
    "humidity": (0.0, 100.0),
    "water_pressure": (0.0, 1000.0),
    "soil_moisture": (0.0, 100.0),     # NEW, %
}
```

`local/subscriber/subscriber.py`, add it to `INSERT_SQL` in both lists:

```python
INSERT_SQL = """
INSERT OR IGNORE INTO sensor_readings
    (msg_id, device, collected_at, received_at, temperature, humidity, water_pressure, soil_moisture)
VALUES (:msg_id, :device, :collected_at, :received_at, :temperature, :humidity, :water_pressure, :soil_moisture)
"""
```

A device that does not send the new value yet is still accepted. The column stays empty
(NULL) for its readings, and a value outside the range is rejected.

**4. Flask API** (`local/web/app.py`): add the column to the three queries.

- `/api/latest`: add `soil_moisture` to the `SELECT` list.
- `/api/history`: add `ROUND(AVG(soil_moisture), 2) AS soil_moisture,` next to the other averages.
- `/api/recent`: add `soil_moisture` to the `SELECT` list.

**5. Chart.js page**

`local/web/templates/index.html`: a card, a chart and a table column:

```html
<div class="card">
  <h2>Soil moisture</h2>
  <p id="soilValue">–</p>
</div>
...
<div class="chart-box"><canvas id="soilChart"></canvas></div>
...
<th>Soil (%)</th>
```

`local/web/static/app.js`:

```javascript
const soilChart = makeChart(
  "soilChart",
  "line",
  "Soil moisture (%)",
  "#2e8b57",
  "%",
);

// inside refresh(), next to the other setData() calls:
setData(
  soilChart,
  labels,
  hist.readings.map((r) => r.soil_moisture),
);
document.getElementById("soilValue").textContent = r
  ? fmt(r.soil_moisture, "%")
  : "–";

// inside the recent-readings row template, before the Delivery cell:
<td>${fmt(row.soil_moisture, "")}</td>;
```

The `.cards` grid has 4 columns. With a 5th card, change it to
`grid-template-columns: repeat(5, 1fr);` in `static/style.css`.

**6. Grafana.** Easiest in the browser: open the dashboard → **Edit** → **Add → Visualization**,
data source `Irrigation (SQLite)`, then:

```sql
SELECT CAST(strftime('%s', collected_at) AS INTEGER) AS time,
       soil_moisture AS "Soil moisture"
FROM sensor_readings
WHERE CAST(strftime('%s', collected_at) AS INTEGER) BETWEEN $__from / 1000 AND $__to / 1000
  AND soil_moisture IS NOT NULL
ORDER BY time
```

- Query options: **Format** = Time series, **Time formatted columns** = `time`.
- Panel: Visualization **Time series**, Unit **Percent (0-100)**, title "Soil moisture history".
- For a current-value tile, duplicate the Humidity **Stat** panel and replace `humidity` with `soil_moisture` in its query.
- Save, then **Export → Export as JSON** and overwrite `grafana/dashboards/irrigation.json`.

**7. Rebuild and restart**

```powershell
docker compose up -d --build subscriber web                     # laptop
docker compose --profile simulator up -d --build pi-sim         # simulated Pi
```

On a real Pi, stop `node.py` (Ctrl+C) and start it again. Readings already waiting in
its buffer are sent as they are, without the new value.

**Checklist**

| Layer      | File                                                        | Change                  |
| ---------- | ----------------------------------------------------------- | ----------------------- |
| Pi         | `pi/sensors.py` (+ copy to `local/common/dummy_sensors.py`) | return the new value    |
| Database   | `local/common/db.py` + one `ALTER TABLE`                    | new column              |
| Subscriber | `validation.py`, `subscriber.py`                            | range + `INSERT_SQL`    |
| Flask      | `web/app.py`                                                | 3 queries               |
| Chart.js   | `index.html`, `static/app.js` (+ `style.css`)               | card, chart, table cell |
| Grafana    | dashboard JSON (via Export)                                 | new panel(s)            |

## Notes

- The Pi's clock must be right (NTP). After a long offline period without internet, check `timedatectl`.
- `allow_anonymous true` and `user: "0"` in Compose are classroom shortcuts. Use a Mosquitto password file and proper folder permissions on a real network.
- Windows laptops: allow inbound TCP 1883 in the firewall so the Pi can connect.
- Change `grafana/grafana:latest` to a fixed version for the term.
