# F1 Data Pipeline — Step 1: Data Inventory & Schema Mapping

**Project:** F1 Intelligence Dashboard (1950–2026)
**Pipeline Stage:** 1 of 8
**Notebook:** `data_inventory.ipynb`
**Status:** ✅ Complete

---

## 1. Dataset Overview

| # | File | Rows | Columns | Memory | Null Columns |
|---|------|------|---------|--------|-------------|
| 1 | circuits.csv | 78 | 9 | 28 KB | ✅ None |
| 2 | constructor_results.csv | 12,898 | 5 | 807 KB | ⚠️ 1 |
| 3 | constructor_standings.csv | 13,664 | 7 | 1,312 KB | ⚠️ 1 |
| 4 | constructors.csv | 214 | 5 | 57 KB | ✅ None |
| 5 | driver_standings.csv | 35,427 | 7 | 3,415 KB | ⚠️ 1 |
| 6 | drivers.csv | 865 | 9 | 361 KB | ⚠️ 2 |
| 7 | lap_times.csv | 618,766 | 6 | 58,614 KB | ✅ None |
| 8 | pit_stops.csv | 22,193 | 7 | 3,296 KB | ⚠️ 2 |
| 9 | qualifying.csv | 11,036 | 9 | 2,065 KB | ⚠️ 3 |
| 10 | races.csv | 1,171 | 18 | 736 KB | ⚠️ 11 |
| 11 | results.csv | 27,304 | 18 | 6,651 KB | ⚠️ 9 |
| 12 | seasons.csv | 77 | 2 | 8 KB | ✅ None |
| 13 | sprint_results.csv | 502 | 17 | 134 KB | ⚠️ 6 |
| 14 | status.csv | 140 | 2 | 9 KB | ✅ None |
| | **TOTAL** | **744,335** | | **~78 MB** | |

---

## 2. Column-Level Inventory

### circuits.csv
| Column | Type | Description |
|--------|------|-------------|
| circuitId | int64 (PK) | Unique circuit identifier |
| circuitRef | object | URL-safe circuit slug |
| name | object | Full circuit name |
| location | object | City |
| country | object | Country |
| lat | float64 | Latitude coordinate |
| lng | float64 | Longitude coordinate |
| alt | int64 | Altitude in metres |
| url | object | Wikipedia URL |

---

### constructor_results.csv
| Column | Type | Notes |
|--------|------|-------|
| constructorResultsId | int64 (PK) | Unique identifier |
| raceId | int64 (FK → races) | |
| constructorId | int64 (FK → constructors) | |
| points | float64 | Points scored in this race |
| status | object | ❌ 99.9% null — drop this column |

---

### constructor_standings.csv
| Column | Type | Notes |
|--------|------|-------|
| constructorStandingsId | int64 (PK) | |
| raceId | int64 (FK → races) | Standings snapshot after this race |
| constructorId | int64 (FK → constructors) | |
| points | float64 | Cumulative season points |
| position | float64 | ⚠️ 1 null — cast to nullable Int64 |
| positionText | object | Handles "D" (DSQ), "E" (excluded) |
| wins | int64 | Cumulative wins |

---

### constructors.csv
| Column | Type | Description |
|--------|------|-------------|
| constructorId | int64 (PK) | |
| constructorRef | object | URL-safe slug |
| name | object | Team name |
| nationality | object | Country of origin |
| url | object | Wikipedia URL |

---

### driver_standings.csv
| Column | Type | Notes |
|--------|------|-------|
| driverStandingsId | int64 (PK) | |
| raceId | int64 (FK → races) | Standings snapshot after this race |
| driverId | int64 (FK → drivers) | |
| points | float64 | Cumulative season points |
| position | float64 | ⚠️ 13 nulls — cast to nullable Int64 |
| positionText | object | |
| wins | int64 | Cumulative wins |

---

### drivers.csv
| Column | Type | Notes |
|--------|------|-------|
| driverId | int64 (PK) | |
| driverRef | object | URL-safe slug |
| number | float64 | ⚠️ 802 nulls — permanent numbers from 2014 only |
| code | object | 3-letter code e.g. "HAM", "VER" — string, NOT integer |
| forename | object | First name |
| surname | object | Last name |
| dob | object | ⚠️ Date of birth — cast to datetime64 |
| nationality | object | |
| url | object | Wikipedia URL |

---

### lap_times.csv
| Column | Type | Description |
|--------|------|-------------|
| raceId | int64 (FK → races) | Composite PK with driverId + lap |
| driverId | int64 (FK → drivers) | |
| lap | int64 | Lap number |
| position | int64 | Track position at end of lap |
| time | object | ⚠️ Duration string "1:27.452" — parse to milliseconds |
| milliseconds | int64 | Lap time in ms — already numeric ✅ |

---

### pit_stops.csv
| Column | Type | Notes |
|--------|------|-------|
| raceId | int64 (FK → races) | |
| driverId | int64 (FK → drivers) | |
| stop | int64 | Stop number (1st, 2nd, 3rd…) |
| lap | int64 | Lap on which stop occurred |
| time | object | Clock time of day — keep as string |
| duration | object | ⚠️ 3 nulls — parse "23.456" → float seconds |
| milliseconds | float64 | ⚠️ 3 nulls — cast to nullable Int64 |

> ⚠️ **Era note:** Pit stop data only exists from **2011 onwards**.

---

### qualifying.csv
| Column | Type | Notes |
|--------|------|-------|
| qualifyId | int64 (PK) | |
| raceId | int64 (FK → races) | |
| driverId | int64 (FK → drivers) | |
| constructorId | int64 (FK → constructors) | |
| number | int64 | Car number |
| position | int64 | Final qualifying position |
| q1 | object | ⚠️ 163 nulls — parse "1:27.452" → milliseconds |
| q2 | object | ⚠️ 4,787 nulls — STRUCTURAL: only top 15 reach Q2 |
| q3 | object | ⚠️ 7,143 nulls — STRUCTURAL: only top 10 reach Q3 |

---

### races.csv
| Column | Type | Notes |
|--------|------|-------|
| raceId | int64 (PK) | |
| year | int64 (FK → seasons) | |
| round | int64 | Round number in season |
| circuitId | int64 (FK → circuits) | |
| name | object | Race name |
| date | object | ⚠️ Cast to datetime64 |
| time | object | Race start clock time — keep as string |
| url | object | Wikipedia URL |
| fp1_date | object | ⚠️ 1,035 nulls — STRUCTURAL: recorded from 2006 only |
| fp1_time | object | Session clock time — keep as string |
| fp2_date | object | ⚠️ STRUCTURAL null — same as fp1 |
| fp2_time | object | Keep as string |
| fp3_date | object | ⚠️ STRUCTURAL null — same as fp1 |
| fp3_time | object | Keep as string |
| quali_date | object | ⚠️ STRUCTURAL null — recorded from 2006 |
| quali_time | object | Keep as string |
| sprint_date | object | ⚠️ 1,141 nulls — STRUCTURAL: sprints from 2021 only |
| sprint_time | object | Keep as string |

---

### results.csv
| Column | Type | Notes |
|--------|------|-------|
| resultId | int64 (PK) | |
| raceId | int64 (FK → races) | |
| driverId | int64 (FK → drivers) | |
| constructorId | int64 (FK → constructors) | |
| number | float64 | ⚠️ 6 nulls — cast to nullable Int64 |
| grid | float64 | ⚠️ 20 nulls — cast to nullable Int64 |
| position | float64 | ⚠️ 10,953 nulls — LOGICAL: null = DNF. Use positionOrder |
| positionText | object | "R"=Retired, "D"=DSQ, "W"=Withdrawn |
| positionOrder | int64 | ✅ Always set — use this for ALL ranking logic |
| points | float64 | Decimals valid (0.5 points exist) |
| laps | int64 | Laps completed |
| time | object | Finishing gap string e.g. "+1:23.1" — keep as string |
| milliseconds | float64 | ⚠️ Nulls for DNFs — cast to nullable Int64 |
| fastestLap | float64 | ⚠️ Nulls — lap number of fastest lap, cast to Int64 |
| rank | float64 | ⚠️ Nulls — fastest lap ranking, cast to Int64 |
| fastestLapTime | object | ⚠️ Duration string — parse to milliseconds |
| fastestLapSpeed | float64 | ✅ Speed in km/h — already correct |
| statusId | int64 (FK → status) | |

---

### seasons.csv
| Column | Type | Description |
|--------|------|-------------|
| year | int64 (PK) | Season year |
| url | object | Wikipedia URL |

---

### sprint_results.csv
| Column | Type | Notes |
|--------|------|-------|
| resultId | int64 (PK) | |
| raceId | int64 (FK → races) | |
| driverId | int64 (FK → drivers) | |
| constructorId | int64 (FK → constructors) | |
| number | int64 | Car number |
| grid | int64 | Starting grid position |
| position | float64 | ⚠️ 15 nulls — cast to nullable Int64 |
| positionText | object | |
| positionOrder | int64 | ✅ Always use for ranking |
| points | int64 | |
| laps | int64 | |
| time | object | Keep as string |
| milliseconds | float64 | ⚠️ Nulls — cast to nullable Int64 |
| fastestLap | float64 | ⚠️ Nulls — cast to Int64 |
| fastestLapTime | object | ⚠️ Parse to milliseconds |
| statusId | int64 (FK → status) | |
| rank | float64 | ⚠️ 364 nulls — STRUCTURAL: not tracked in sprints |

> ⚠️ **Era note:** Sprint results only exist from **2021 onwards**.

---

### status.csv
| Column | Type | Description |
|--------|------|-------------|
| statusId | int64 (PK) | |
| status | object | 140 codes: "Finished", "Engine", "Accident" etc. |

---

## 3. Schema Map — Entity Relationship

```
seasons (year)
    └── races (raceId, year, circuitId)
            ├── circuits (circuitId)
            ├── results (raceId, driverId, constructorId, statusId)
            │       ├── drivers (driverId)
            │       ├── constructors (constructorId)
            │       └── status (statusId)
            ├── lap_times (raceId, driverId)
            ├── pit_stops (raceId, driverId)
            ├── qualifying (raceId, driverId, constructorId)
            ├── sprint_results (raceId, driverId, constructorId, statusId)
            ├── driver_standings (raceId, driverId)
            ├── constructor_standings (raceId, constructorId)
            └── constructor_results (raceId, constructorId)
```

**Central hub table:** `races` — every analytical table links through it.
**Core fact table:** `results` — main record of every driver's race outcome.

![F1 Database ERD](../images/F1_ERD_v2.png)

---

## 4. Foreign Key Validation

All 19 foreign key relationships tested across all 14 tables.

**Result: ✅ 100% integrity — zero orphaned records.**

---

## 5. Dataset Scope

| Dimension | Value |
|-----------|-------|
| Year range | 1950 – 2026 |
| Total races | 1,171 |
| Total drivers | 865 |
| Total constructors | 214 |
| Total circuits | 78 |
| Status codes | 140 |
| Lap time records | 618,766 |
| Pit stop records | 22,193 (2011–2026 only) |
| Sprint events | ~30 (2021–2026 only) |
| **Total rows** | **744,335** |

---

## 6. Era Boundaries

Three critical era boundaries affect every analysis in this project.
Every model and query must respect these cutoffs.

| Era Boundary | Year | Impact |
|-------------|------|--------|
| Pit stop data begins | 2011 | No pit strategy analysis before 2011 |
| FP session data begins | 2006 | No practice data before 2006 |
| Sprint races begin | 2021 | Sprint results only in recent seasons |
| Permanent driver numbers | 2014 | drivers.number null before 2014 |
| Q1/Q2/Q3 format begins | 2006 | qualifying.q2/q3 structural nulls before 2006 |
| Fastest lap time recorded | ~1996 | fastestLapTime null before ~1996 |
| Fastest lap speed recorded | ~2004 | fastestLapSpeed null before ~2004 |

---

## 7. What's Next — Step 2

| Task | Description | Status |
|------|-------------|--------|
| Task 1 | Type audit — identify all wrong types | ✅ Done |
| Task 2 | Cast date columns to datetime64 | 🔜 Next |
| Task 3 | Parse time strings to milliseconds | 🔜 Pending |
| Task 4 | Cast float columns to nullable Int64 | 🔜 Pending |

---

*F1 Data Pipeline · Step 1 · Data Inventory & Schema Map · 2026*