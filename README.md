# Tracking-Device Survival Analysis — Fleetoo

A Kaplan–Meier survival analysis of **GPS/telematics device transmission continuity** across a
fleet of ~4,250 installed devices. It answers a single operational question:

> **How long does a tracking device keep transmitting after it is installed on a vehicle —
> and which of them have gone silent?**

The result is a single, self-contained, **anonymized** HTML page (`index.html`) — inline SVG
charts, no external libraries, no back-end. Drop it on GitHub Pages and it renders as an
interactive report.

**Live page:** 
`https://ameenelzaki96.github.io/Survival-Analysis`

---

## Key findings (snapshot: 2026-08-20)

| Metric | Value |
|--------|-------|
| Devices installed | 4,250 |
| Failures (silent > 14 days) | 413 |
| Still transmitting (censored) | 3,260 |
| **Never transmitted at all** (excluded) | **577** |
| Median survival | Not reached (> 50% still transmitting) |
| Survival at 30 / 90 / 180 / 365 days | 95.4% / 90.9% / 81.2% / 81.2% |

Two headlines: **577 devices never sent a single byte** after being assigned (an install/SIM
problem, not a wear-out failure), and observed failures **cluster in the first ~150 days**.

---

## Method

Survival analysis measures time-to-event while correctly handling subjects that **haven't had
the event yet** (censoring) — the reason we don't just average failure times.

- **Subject** — one device installed on a vehicle.
- **t = 0** — `assignedToVehicleAt` (install date).
- **Event ("failure")** — device silent for **> 14 days**. Time-to-failure = install → last signal.
- **Censored** — still transmitting today; counted as "survived at least this long."
- **Estimator** — Kaplan–Meier `S(t)` with **Greenwood** 95% confidence intervals.

"Silence" measures **transmission continuity**, not strictly hardware failure — a parked
vehicle, a suspended SIM, or an uninstalled unit all look the same. That is the right signal
for proactive maintenance.

---

## Data pipeline

Two production data sources (read-only):

1. **PostgreSQL** — device roster: which devices are assigned to a vehicle, their install date,
   and the owning client. (Client names are used only to rank, then **anonymized** to
   `Client 1…N` in the published page.)
2. **ClickHouse** (`device_info`, ~774M rows) — the **true last-seen** timestamp per device
   (`max(created_at)` grouped by IMEI). This is the authoritative "last transmission" signal.

The two are joined by device IMEI.

---

## The code

The whole thing is one Python script: pull → compute Kaplan–Meier → render SVG → write
`index.html`. Credentials are read from environment variables — **never hard-code them**.

```bash
export PG_URL="postgresql://<user>:<pass>@<host>:5432/<db>?sslmode=require"
export CH_HOST="<clickhouse-host>"
export CH_USER="<user>"
export CH_PASS="<pass>"
python build.py
```

### 1 — Device roster (PostgreSQL)

```python
import os, psycopg
with psycopg.connect(os.environ["PG_URL"]) as conn:
    devices = conn.execute('''
        SELECT d."imei"::text,
               d."assignedToVehicleAt",
               COALESCE(NULLIF(TRIM(c."arName"::text), ''), c."name"::text) AS client
        FROM "Device" d
        LEFT JOIN "Company" c ON c.id = d."companyId"
        WHERE d."deletedAt" IS NULL          -- active devices only
          AND d."vehicleId" IS NOT NULL       -- must be assigned to a vehicle
          AND d."assignedToVehicleAt" IS NOT NULL
    ''').fetchall()
```

### 2 — True last-seen per device (ClickHouse)

```python
import clickhouse_connect
ch = clickhouse_connect.get_client(
    host=os.environ["CH_HOST"], port=8443,
    username=os.environ["CH_USER"], password=os.environ["CH_PASS"],
    database="fleetoo", secure=True, send_receive_timeout=600)

last_seen = {str(imei): ts for imei, ts in
    ch.query("SELECT imei, max(created_at) FROM device_info GROUP BY imei").result_rows}
```

### 3 — Build the cohort (event vs. censored)

```python
import datetime as dt
NOW, SILENCE = dt.datetime.now(dt.UTC).replace(tzinfo=None), 14
cohort, never = [], 0
for imei, assigned, client in devices:
    assigned = assigned.replace(tzinfo=None)
    last = last_seen.get(imei)
    if last is None:                      # assigned but never transmitted -> excluded
        never += 1; continue
    last = last.replace(tzinfo=None)
    days_silent = (NOW - last).days
    event = 1 if days_silent > SILENCE else 0            # 1 = failed, 0 = censored
    dur = (last - assigned).days if event else (NOW - assigned).days
    if dur < 0:                            # telemetry predates this assignment -> drop
        continue
    cohort.append({"client": client, "event": event, "dur": dur})
```

### 4 — Kaplan–Meier with Greenwood variance

```python
import math
from collections import defaultdict
ev, ce = defaultdict(int), defaultdict(int)
for c in cohort:
    (ev if c["event"] else ce)[c["dur"]] += 1

n = at_risk = len(cohort)
S, var_sum, KM = 1.0, 0.0, [(0, n, 0, 0, 1.0, 1.0, 1.0)]   # t, n_at_risk, d, c, S, lo, hi
for t in sorted(set(ev) | set(ce)):
    d, c = ev[t], ce[t]
    if d and at_risk:
        S *= 1 - d / at_risk                               # KM product
        if at_risk > d:
            var_sum += d / (at_risk * (at_risk - d))        # Greenwood
        se = S * math.sqrt(var_sum)
        KM.append((t, at_risk, d, c, S, max(0.0, S - 1.96*se), min(1.0, S + 1.96*se)))
    at_risk -= d + c
```

`S_at(t)` (step-function lookup) gives survival at 30/90/180/365 days; the median is the first
`t` where `S(t) ≤ 0.5` (here: not reached).

### 5 — Render + anonymize

The KM curve, the time-to-failure histogram, and the per-client bar chart are emitted as inline
**SVG** into a single HTML string (no chart library). In the published page, client names are
**replaced by `Client 1…N`**, ranked by silent-device count — no real client identity leaves
the analysis.

---

## Privacy

This published page is **anonymized**: client identities appear only as `Client 1…N`; no
IMEIs, plates, VINs, or company names are included. (The internal Excel workbook used by the
operations team keeps real identifiers — it is not part of this repo.)

## Caveats

- **Silence ≠ hardware failure** — see Method.
- **Right-censoring / observation window** — many devices were installed recently, so the curve
  beyond ~180 days rests on fewer devices; treat the long tail as lower-confidence.
- Figures are a **single point-in-time snapshot** and shift slightly as live telemetry arrives.
