# ChickMonitor — InfluxDB & Grafana Setup Guide

> **Purpose:** Configure egg production analytics for the ChickMonitor system.  
> **Scope:** InfluxDB (time-series storage) + Grafana (visualization), both running as Home Assistant add-ons. No separate server required.  
> **Assumption:** Home Assistant OS or Supervised installation with Add-on Store access.

---

## Table of Contents

1. [Install and Configure InfluxDB](#1-install-and-configure-influxdb)
2. [Configure HA InfluxDB Integration](#2-configure-ha-influxdb-integration)
3. [Install and Configure Grafana](#3-install-and-configure-grafana)
4. [Build Grafana Dashboards](#4-build-grafana-dashboards)
5. [Import Historical Data](#5-import-historical-data)
6. [Midnight Reset Automation](#6-midnight-reset-automation)
7. [Backup and Maintenance](#7-backup-and-maintenance)

---

## 1. Install and Configure InfluxDB

### 1.1 Install the Add-on

1. In Home Assistant, go to **Settings → Add-ons → Add-on Store**.
2. Search for **InfluxDB** and select the official add-on by Home Assistant.
3. Click **Install** and wait for it to complete.
4. On the **Configuration** tab, leave defaults — the add-on binds to port `8086` on the HA host.
5. Click **Start**, then enable **Start on boot** and **Watchdog**.

### 1.2 Create the Database

1. Open the InfluxDB admin UI: `http://<your-ha-ip>:8086`.
2. Complete the initial setup wizard:
   - **Username:** `admin` (or your preference)
   - **Password:** choose a strong password and save it
   - **Organization:** `chickmonitor`
   - **Bucket:** `chickmonitor` ← this is the database name used throughout this guide
3. Under **Load Data → Buckets**, find the `chickmonitor` bucket.
4. Click the bucket → **Settings** → set **Data Retention** to **Forever** (no expiry).

> **Note:** If you are using InfluxDB 1.x (the older HA add-on), create the database via the Chronograf UI or the CLI:
> ```
> CREATE DATABASE chickmonitor
> CREATE RETENTION POLICY "forever" ON "chickmonitor" DURATION INF REPLICATION 1 DEFAULT
> ```

### 1.3 Create a Read-Only Token (for Grafana)

1. In the InfluxDB UI, go to **Load Data → API Tokens**.
2. Click **Generate API Token → Read/Write API Token**.
3. Give it the name `grafana-reader`.
4. Under **Read**, select the `chickmonitor` bucket.
5. Leave **Write** empty.
6. Click **Save** and copy the token — you will need it in Section 3.

### 1.4 Create a Write Token (for HA integration)

Repeat the above process:
- Name: `ha-writer`
- Write access to `chickmonitor` bucket only
- Copy and save the token — you will need it in Section 2.

---

## 2. Configure HA InfluxDB Integration

The HA `influxdb:` integration forwards entity state changes to InfluxDB automatically. Only `number.coop_egg_count` needs to be forwarded — other entities are optional.

### 2.1 Add to `configuration.yaml`

```yaml
influxdb:
  host: localhost          # InfluxDB runs on the same host as HA
  port: 8086
  database: chickmonitor   # InfluxDB 1.x: database name
  # For InfluxDB 2.x, use these instead of host/port/database:
  # api_version: 2
  # token: <your ha-writer token>
  # organization: chickmonitor
  # bucket: chickmonitor
  ssl: false
  verify_ssl: false
  include:
    entities:
      - number.coop_egg_count
  # Optional — include other ChickMonitor entities if you want full history in InfluxDB:
  # - sensor.coop_temperature
  # - sensor.coop_humidity
  # - binary_sensor.coop_nest_occupied
  # - sensor.coop_door_state
```

### 2.2 Restart and Verify

1. Restart Home Assistant (**Developer Tools → Restart**).
2. Change the egg count on the device (turn the encoder) or from the HA dashboard.
3. In InfluxDB UI, go to **Explore** (InfluxDB 2.x) or **Data Explorer** (1.x).
4. Query for `number.coop_egg_count` measurements in the `chickmonitor` bucket.
5. Confirm data points appear with the correct timestamp.

---

## 3. Install and Configure Grafana

### 3.1 Install the Add-on

1. In the Add-on Store, search for **Grafana**.
2. Install the official Grafana add-on.
3. Start it and enable **Start on boot** and **Watchdog**.
4. Access Grafana at `http://<your-ha-ip>:3000`.
5. Log in with default credentials `admin` / `admin` and set a new password.

### 3.2 Add InfluxDB as a Data Source

1. In Grafana, go to **Connections → Data Sources → Add data source**.
2. Select **InfluxDB**.
3. Configure:

**For InfluxDB 2.x (Flux query language):**

| Field | Value |
|-------|-------|
| Query Language | Flux |
| URL | `http://localhost:8086` |
| Organization | `chickmonitor` |
| Token | *(paste the `grafana-reader` token)* |
| Default Bucket | `chickmonitor` |

**For InfluxDB 1.x (InfluxQL):**

| Field | Value |
|-------|-------|
| Query Language | InfluxQL |
| URL | `http://localhost:8086` |
| Database | `chickmonitor` |
| User | `admin` |
| Password | *(your InfluxDB password)* |

4. Click **Save & Test** — confirm it returns "Data source is working."

---

## 4. Build Grafana Dashboards

Create a new dashboard (**Dashboards → New → New Dashboard**) and add panels using the queries below. Each panel uses the **InfluxDB** data source configured above.

> The queries below use **InfluxQL** syntax (InfluxDB 1.x). If you are on InfluxDB 2.x, translate to **Flux** — the logic is identical, the syntax differs. Flux equivalents are noted where relevant.

---

### Panel 1 — Daily Egg Count (Bar Chart)

**Purpose:** Eggs per day for the last 90 days.

**Query (InfluxQL):**
```sql
SELECT max("value")
FROM "number.coop_egg_count"
WHERE $timeFilter
GROUP BY time(1d) fill(0)
```

**Panel settings:**
- Visualization: **Bar chart**
- Time range: Last 90 days
- X-axis: time, 1-day buckets
- Y-axis: label "Eggs", min 0

> `max(value)` per day gives the day's total because the counter only increases during the day and resets to 0 at midnight. Days with no data fill as 0.

---

### Panel 2 — Average Eggs by Day of Week

**Purpose:** Which days of the week are most productive.

**Query (InfluxQL):**
```sql
SELECT mean("daily_max")
FROM (
  SELECT max("value") AS "daily_max"
  FROM "number.coop_egg_count"
  WHERE $timeFilter
  GROUP BY time(1d)
)
GROUP BY time(7d)
```

> InfluxQL does not natively group by weekday. For a true Mon–Sun breakdown, use **Grafana's Transform** tab: run the daily query above, then apply **Group by → Day of week** using a **Transformation → Group By** step, or use Flux which supports `date.weekDay()` natively.

**Flux alternative (InfluxDB 2.x):**
```flux
from(bucket: "chickmonitor")
  |> range(start: v.timeRangeStart, stop: v.timeRangeStop)
  |> filter(fn: (r) => r._measurement == "number.coop_egg_count")
  |> aggregateWindow(every: 1d, fn: max, createEmpty: false)
  |> map(fn: (r) => ({ r with weekday: date.weekDay(t: r._time) }))
  |> group(columns: ["weekday"])
  |> mean(column: "_value")
```

**Panel settings:**
- Visualization: **Bar chart**
- X-axis: weekday (0=Sun through 6=Sat)
- Y-axis: label "Avg Eggs"

---

### Panel 3 — Monthly Totals (Bar Chart, Year-over-Year)

**Purpose:** Total eggs per calendar month, overlaid across years.

**Query (InfluxQL):**
```sql
SELECT sum("daily_max")
FROM (
  SELECT max("value") AS "daily_max"
  FROM "number.coop_egg_count"
  WHERE $timeFilter
  GROUP BY time(1d)
)
GROUP BY time(30d)
```

> For true calendar-month grouping (not 30-day windows), use Flux `date.truncate(t: r._time, unit: 1mo)` or group by month in a transformation.

**Panel settings:**
- Visualization: **Bar chart** or **Time series**
- Time range: Last 2 years (or All time)
- Y-axis: label "Total Eggs"

---

### Panel 4 — Yearly Production Curve (Line Chart)

**Purpose:** Seasonal trend showing summer peak and winter slowdown.

**Query (InfluxQL):**
```sql
SELECT sum("daily_max")
FROM (
  SELECT max("value") AS "daily_max"
  FROM "number.coop_egg_count"
  WHERE $timeFilter
  GROUP BY time(1d)
)
GROUP BY time(7d) fill(0)
```

**Panel settings:**
- Visualization: **Time series** (line chart)
- Time range: Last 12–24 months
- Smooth line enabled
- Y-axis: label "Eggs (weekly total)"

---

### Panel 5 — All-Time Stats (Stat Panels)

Add individual **Stat** panels for at-a-glance numbers:

| Stat | Query |
|------|-------|
| All-time total | `SELECT sum("daily_max") FROM (SELECT max("value") AS "daily_max" FROM "number.coop_egg_count" GROUP BY time(1d))` |
| Best single day | `SELECT max("daily_max") FROM (SELECT max("value") AS "daily_max" FROM "number.coop_egg_count" GROUP BY time(1d))` |
| This month total | Same as all-time total query, filtered to current month |
| Today's count | `SELECT last("value") FROM "number.coop_egg_count" WHERE time > now() - 1d` |

---

### Saving the Dashboard

1. Click **Save dashboard** (floppy disk icon, top right).
2. Name it **ChickMonitor — Egg Production**.
3. Optionally set it as the Grafana home dashboard under **Preferences**.

---

## 5. Import Historical Data

If you have prior egg count records (e.g., a notebook or spreadsheet), you can load them into InfluxDB so all dashboards reflect your full history.

### 5.1 Prepare the CSV

Create a file named `egg_history.csv` with one row per day:

```csv
date,eggs
2023-01-01,4
2023-01-02,6
2023-01-03,3
...
```

- `date`: ISO 8601 format, `YYYY-MM-DD`
- `eggs`: integer, daily total
- Days with no data can be omitted (they will fill as 0 in Grafana)

### 5.2 Import with Python

Install the InfluxDB client:

```bash
pip install influxdb-client   # InfluxDB 2.x
# or
pip install influxdb          # InfluxDB 1.x
```

**Import script (InfluxDB 2.x):**

```python
import csv
from datetime import datetime, timezone
from influxdb_client import InfluxDBClient, Point, WritePrecision
from influxdb_client.client.write_api import SYNCHRONOUS

# --- Configuration ---
INFLUX_URL    = "http://<your-ha-ip>:8086"
INFLUX_TOKEN  = "<your ha-writer token>"
INFLUX_ORG    = "chickmonitor"
INFLUX_BUCKET = "chickmonitor"
CSV_FILE      = "egg_history.csv"
# ---------------------

client = InfluxDBClient(url=INFLUX_URL, token=INFLUX_TOKEN, org=INFLUX_ORG)
write_api = client.write_api(write_options=SYNCHRONOUS)

with open(CSV_FILE, newline="") as f:
    reader = csv.DictReader(f)
    points = []
    for row in reader:
        # Use 23:59:00 UTC so max(value) per day query captures the day's final count
        ts = datetime.strptime(row["date"], "%Y-%m-%d").replace(
            hour=23, minute=59, second=0, tzinfo=timezone.utc
        )
        point = (
            Point("number.coop_egg_count")      # must match the HA measurement name
            .field("value", int(row["eggs"]))
            .time(ts, WritePrecision.SECONDS)
        )
        points.append(point)

    write_api.write(bucket=INFLUX_BUCKET, record=points)
    print(f"Imported {len(points)} days of historical data.")

client.close()
```

**Import script (InfluxDB 1.x):**

```python
import csv
from datetime import datetime, timezone
from influxdb import InfluxDBClient

INFLUX_HOST = "<your-ha-ip>"
INFLUX_PORT = 8086
INFLUX_DB   = "chickmonitor"
CSV_FILE    = "egg_history.csv"

client = InfluxDBClient(host=INFLUX_HOST, port=INFLUX_PORT, database=INFLUX_DB)

points = []
with open(CSV_FILE, newline="") as f:
    reader = csv.DictReader(f)
    for row in reader:
        ts = datetime.strptime(row["date"], "%Y-%m-%d").replace(
            hour=23, minute=59, second=0, tzinfo=timezone.utc
        )
        points.append({
            "measurement": "number.coop_egg_count",
            "time": ts.isoformat(),
            "fields": {"value": float(row["eggs"])}
        })

client.write_points(points)
print(f"Imported {len(points)} days of historical data.")
```

### 5.3 Verify the Import

1. Run the script from your workstation (InfluxDB port 8086 must be reachable on your local network).
2. In the InfluxDB UI, query the `chickmonitor` bucket and confirm historical timestamps appear.
3. Open Grafana and extend the time range to cover your historical dates — data should populate immediately.

> **Idempotency:** Running the script twice will overwrite the same timestamps with the same values (InfluxDB uses last-write-wins per timestamp). Re-running with corrected data is safe.

---

## 6. Midnight Reset Automation

The following HA automation resets the egg count to 0 at midnight each day. The previous day's total is already captured in InfluxDB before the reset fires.

```yaml
alias: ChickMonitor — Reset Egg Count at Midnight
description: >
  Resets number.coop_egg_count to 0 at midnight.
  InfluxDB captures the day's final max value before this fires.
trigger:
  - platform: time
    at: "00:00:00"
action:
  - service: number.set_value
    target:
      entity_id: number.coop_egg_count
    data:
      value: 0
mode: single
```

Add this automation via **Settings → Automations → New Automation → Edit in YAML**.

---

## 7. Backup and Maintenance

### InfluxDB Backup

HA snapshots (full backups) include add-on data, so InfluxDB data is included in your regular HA backup. To create a standalone InfluxDB backup:

**InfluxDB 2.x:**
```bash
# Run from the HA host shell (or SSH add-on)
influx backup /backup/influxdb-$(date +%Y%m%d) --host http://localhost:8086 --token <admin-token>
```

**InfluxDB 1.x:**
```bash
influxd backup -portable /backup/influxdb-$(date +%Y%m%d)
```

Schedule this as a shell command in HA or run it before any major system changes.

### Grafana Dashboard Backup

Export dashboards as JSON: **Dashboard menu → Share → Export → Save to file**. Commit these JSON files to the ChickMonitor git repository alongside the project plan.

### Ongoing Maintenance

- No regular maintenance is required — InfluxDB retention is set to Forever.
- If the HA host is replaced, restore from an HA snapshot and all data is preserved.
- Grafana dashboards can be re-imported from the saved JSON files.

---

*Document version 1.0 — Initial setup guide for InfluxDB + Grafana egg production analytics. Covers InfluxDB 1.x and 2.x. Includes historical import scripts for both client versions.*