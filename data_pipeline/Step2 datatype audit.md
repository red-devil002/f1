# F1 Data Pipeline — Step 2: Data Type Audit
**Project:** F1 Intelligence Dashboard (1950–2026)
**Pipeline Stage:** 2 of 8
**Notebook:** `Standardisation.ipynb`
**Status:** ✅ Task 1 Complete — Audit Done

---

## What This Document Is

Before transforming any data, every column across all 14 tables was
audited for correct data types. This document records what pandas
assigned on load, what the column should actually be, and why.

This audit drives every casting decision in Tasks 2, 3, and 4.

---

## The 4 Patterns We Looked For

| Pattern | Example | Problem |
|---------|---------|---------|
| Date stored as string | `races.date = object` | Can't do date arithmetic |
| Duration stored as string | `lap_times.time = object` | Can't calculate average speed |
| Integer stored as float | `results.grid = float64` | Nulls forced pandas to use float |
| String mislabelled | `results.positionText = object` | Should be explicit string type |

---

## Full Audit Table

### circuits

| Column | Current Type | Should Be | Action Required |
|--------|-------------|-----------|-----------------|
| circuitId | int64 | int64 | ✅ None |
| circuitRef | object | string | Low priority |
| name | object | string | Low priority |
| location | object | string | Low priority |
| country | object | string | Low priority |
| lat | float64 | float64 | ✅ None |
| lng | float64 | float64 | ✅ None |
| alt | int64 | int64 | ✅ None |
| url | object | object | ✅ Drop in Step 6 |

---

### constructor_results

| Column | Current Type | Should Be | Action Required |
|--------|-------------|-----------|-----------------|
| constructorResultsId | int64 | int64 | ✅ None |
| raceId | int64 | int64 | ✅ None |
| constructorId | int64 | int64 | ✅ None |
| points | float64 | float64 | ✅ None — decimals valid (0.5 pts exist) |
| status | object | — | ❌ Drop entirely — 99.9% null |

---

### constructor_standings

| Column | Current Type | Should Be | Action Required |
|--------|-------------|-----------|-----------------|
| constructorStandingsId | int64 | int64 | ✅ None |
| raceId | int64 | int64 | ✅ None |
| constructorId | int64 | int64 | ✅ None |
| points | float64 | float64 | ✅ None |
| position | float64 | Int64 | ⚠️ Float due to 1 null — cast to nullable Int64 |
| positionText | object | string | Low priority |
| wins | int64 | int64 | ✅ None |

---

### constructors

| Column | Current Type | Should Be | Action Required |
|--------|-------------|-----------|-----------------|
| constructorId | int64 | int64 | ✅ None |
| constructorRef | object | string | Low priority |
| name | object | string | Low priority |
| nationality | object | string | Low priority |
| url | object | object | ✅ Drop in Step 6 |

---

### driver_standings

| Column | Current Type | Should Be | Action Required |
|--------|-------------|-----------|-----------------|
| driverStandingsId | int64 | int64 | ✅ None |
| raceId | int64 | int64 | ✅ None |
| driverId | int64 | int64 | ✅ None |
| points | float64 | float64 | ✅ None |
| position | float64 | Int64 | ⚠️ Float due to 13 nulls — cast to nullable Int64 |
| positionText | object | string | Low priority |
| wins | int64 | int64 | ✅ None |

---

### drivers

| Column | Current Type | Should Be | Action Required |
|--------|-------------|-----------|-----------------|
| driverId | int64 | int64 | ✅ None |
| driverRef | object | string | Low priority |
| number | float64 | Int64 | ⚠️ Float due to 802 nulls — cast to nullable Int64 |
| code | object | string | ✅ None — values are "HAM", "VER" etc. NOT integer |
| forename | object | string | Low priority |
| surname | object | string | Low priority |
| dob | object | datetime64 | ⚠️ Cast with pd.to_datetime() |
| nationality | object | string | Low priority |
| url | object | object | ✅ Drop in Step 6 |

> ⚠️ `drivers.code` contains 3-letter strings like "HAM", "VER", "LEC"
> — it is **not** an integer and must never be cast to one.

---

### lap_times

| Column | Current Type | Should Be | Action Required |
|--------|-------------|-----------|-----------------|
| raceId | int64 | int64 | ✅ None |
| driverId | int64 | int64 | ✅ None |
| lap | int64 | int64 | ✅ None |
| position | int64 | int64 | ✅ None |
| time | object | int64 (ms) | ⚠️ Parse "1:27.452" → milliseconds |
| milliseconds | int64 | int64 | ✅ Already numeric — use directly |

> `lap_times.time` is a **duration string**, not a datetime.
> Convert to milliseconds for all calculations.
> `milliseconds` column already exists and is clean — use it directly.

---

### pit_stops

| Column | Current Type | Should Be | Action Required |
|--------|-------------|-----------|-----------------|
| raceId | int64 | int64 | ✅ None |
| driverId | int64 | int64 | ✅ None |
| stop | int64 | int64 | ✅ None |
| lap | int64 | int64 | ✅ None |
| time | object | string | ✅ Clock time of day — keep as string |
| duration | object | float64 | ⚠️ Parse "23.456" → float seconds |
| milliseconds | float64 | Int64 | ⚠️ Float due to 3 nulls — cast to nullable Int64 |

---

### qualifying

| Column | Current Type | Should Be | Action Required |
|--------|-------------|-----------|-----------------|
| qualifyId | int64 | int64 | ✅ None |
| raceId | int64 | int64 | ✅ None |
| driverId | int64 | int64 | ✅ None |
| constructorId | int64 | int64 | ✅ None |
| number | int64 | int64 | ✅ None |
| position | int64 | int64 | ✅ None |
| q1 | object | int64 (ms) | ⚠️ Parse "1:27.452" → milliseconds |
| q2 | object | int64 (ms) | ⚠️ Parse — preserve nulls (structural) |
| q3 | object | int64 (ms) | ⚠️ Parse — preserve nulls (structural) |

---

### races

| Column | Current Type | Should Be | Action Required |
|--------|-------------|-----------|-----------------|
| raceId | int64 | int64 | ✅ None |
| year | int64 | int64 | ✅ None |
| round | int64 | int64 | ✅ None |
| circuitId | int64 | int64 | ✅ None |
| name | object | string | Low priority |
| date | object | datetime64 | ⚠️ Cast with pd.to_datetime() |
| time | object | string | ✅ Race start clock time — keep as string |
| url | object | object | ✅ Drop in Step 6 |
| fp1_date | object | datetime64 | ⚠️ Cast — preserve nulls (structural) |
| fp1_time | object | string | ✅ Session clock time — keep as string |
| fp2_date | object | datetime64 | ⚠️ Cast — preserve nulls (structural) |
| fp2_time | object | string | ✅ Keep as string |
| fp3_date | object | datetime64 | ⚠️ Cast — preserve nulls (structural) |
| fp3_time | object | string | ✅ Keep as string |
| quali_date | object | datetime64 | ⚠️ Cast — preserve nulls (structural) |
| quali_time | object | string | ✅ Keep as string |
| sprint_date | object | datetime64 | ⚠️ Cast — preserve nulls (structural) |
| sprint_time | object | string | ✅ Keep as string |

> ⚠️ `races.time`, `fp1_time` etc. are **clock times** (e.g. "13:00:00")
> not durations and not datetimes. Keep as string — they are rarely
> needed for analysis.

---

### results

| Column | Current Type | Should Be | Action Required |
|--------|-------------|-----------|-----------------|
| resultId | int64 | int64 | ✅ None |
| raceId | int64 | int64 | ✅ None |
| driverId | int64 | int64 | ✅ None |
| constructorId | int64 | int64 | ✅ None |
| number | float64 | Int64 | ⚠️ Float due to 6 nulls — cast to nullable Int64 |
| grid | float64 | Int64 | ⚠️ Float due to 20 nulls — cast to nullable Int64 |
| position | float64 | Int64 | ⚠️ Float due to 10,953 nulls (DNFs) — cast to nullable Int64 |
| positionText | object | string | Low priority |
| positionOrder | int64 | int64 | ✅ None — always use this for ranking |
| points | float64 | float64 | ✅ None — decimals valid |
| laps | int64 | int64 | ✅ None |
| time | object | string | ✅ Finishing gap string e.g. "+1:23.1" — keep as string |
| milliseconds | float64 | Int64 | ⚠️ Float due to nulls (DNFs) — cast to nullable Int64 |
| fastestLap | float64 | Int64 | ⚠️ Float due to nulls — cast to nullable Int64 |
| rank | float64 | Int64 | ⚠️ Float due to nulls — cast to nullable Int64 |
| fastestLapTime | object | int64 (ms) | ⚠️ Parse "1:27.452" → milliseconds |
| fastestLapSpeed | float64 | float64 | ✅ None — speed in km/h, decimals valid |
| statusId | int64 | int64 | ✅ None |

> ⚠️ `results.time` is a finishing gap string like `"+1:23.456"` or
> `"1:34:56.789"` — it is NOT a datetime and NOT a simple duration.
> Keep as string for display. Use `milliseconds` column for calculations.
>
> ⚠️ `results.fastestLapSpeed` is already `float64` and correct —
> it contains speed values like `218.3` km/h. It is **not** a datetime.

---

### seasons

| Column | Current Type | Should Be | Action Required |
|--------|-------------|-----------|-----------------|
| year | int64 | int64 | ✅ None |
| url | object | object | ✅ Drop in Step 6 |

---

### sprint_results

| Column | Current Type | Should Be | Action Required |
|--------|-------------|-----------|-----------------|
| resultId | int64 | int64 | ✅ None |
| raceId | int64 | int64 | ✅ None |
| driverId | int64 | int64 | ✅ None |
| constructorId | int64 | int64 | ✅ None |
| number | int64 | int64 | ✅ None |
| grid | int64 | int64 | ✅ None |
| position | float64 | Int64 | ⚠️ Float due to 15 nulls — cast to nullable Int64 |
| positionText | object | string | Low priority |
| positionOrder | int64 | int64 | ✅ None |
| points | int64 | int64 | ✅ None |
| laps | int64 | int64 | ✅ None |
| time | object | string | ✅ Keep as string |
| milliseconds | float64 | Int64 | ⚠️ Float due to nulls — cast to nullable Int64 |
| fastestLap | float64 | Int64 | ⚠️ Float due to nulls — cast to nullable Int64 |
| fastestLapTime | object | int64 (ms) | ⚠️ Parse "1:27.452" → milliseconds |
| statusId | int64 | int64 | ✅ None |
| rank | float64 | Int64 | ⚠️ Float due to nulls — cast to nullable Int64 |

---

### status

| Column | Current Type | Should Be | Action Required |
|--------|-------------|-----------|-----------------|
| statusId | int64 | int64 | ✅ None |
| status | object | string | Low priority |

---

## Action Summary

| Action | Columns Affected | Task |
|--------|-----------------|------|
| `pd.to_datetime()` | `races.date`, all `_date` columns, `drivers.dob` | Task 2 |
| Parse time string → ms | `lap_times.time`, `qualifying.q1/q2/q3`, `results.fastestLapTime`, `sprint_results.fastestLapTime`, `pit_stops.duration` | Task 3 |
| `float64` → `Int64` | `position`, `grid`, `number`, `milliseconds`, `fastestLap`, `rank` across multiple tables | Task 4 |
| Drop column | `constructor_results.status` | Step 6 |
| Drop column | All `url` columns | Step 6 |

---

## Key Correction — Common Mistakes

| Column | Wrong Assumption | Correct Type | Reason |
|--------|-----------------|--------------|--------|
| `drivers.code` | int64 | string | Contains "HAM", "VER" — text, not numbers |
| `results.time` | datetime | string | Finishing gap e.g. "+1:23.1" — not a point in time |
| `results.fastestLapTime` | datetime | int64 (ms) | Duration string — parse to milliseconds |
| `results.fastestLapSpeed` | datetime | float64 | Speed in km/h — already correct |
| `races.*_time` | datetime | string | Clock times e.g. "13:00:00" — keep as string |

---

## What's Next — Task 2

With the audit complete, Task 2 applies the first set of fixes:
casting all date columns from strings to `datetime64` using
`pd.to_datetime()`.

```python
# Preview of what Task 2 looks like
date_cols = ['date', 'fp1_date', 'fp2_date', 'fp3_date',
             'quali_date', 'sprint_date']

for col in date_cols:
    dfs['races'][col] = pd.to_datetime(dfs['races'][col], errors='coerce')
```

---

*F1 Data Pipeline · Step 2 · Task 1 — Type Audit · 2026*