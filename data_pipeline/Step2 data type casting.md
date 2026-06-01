# F1 Data Pipeline — Step 2: Data Type Casting & Standardisation

**Project:** F1 Intelligence Dashboard (1950–2026)
**Pipeline Stage:** 2 of 8
**Notebook:** `Standardisation.ipynb`
**Status:** ✅ Complete

---

## Why This Step Exists

When pandas loads a CSV it *guesses* each column's type. Many guesses
are wrong — dates load as strings, lap times load as text, and integer
columns load as floats because of nulls. Running analysis on wrong
types either crashes or silently produces wrong answers.

Step 2 corrects every type so the data is analysis-ready.

---

## The Four Tasks

| Task | Goal | Status |
|------|------|--------|
| Task 1 | Audit every column for wrong types | ✅ |
| Task 2 | Cast date columns to `datetime64` | ✅ |
| Task 3 | Parse time strings to milliseconds | ✅ |
| Task 4 | Cast float columns to nullable `Int64` | ✅ |

---

## Task 1 — Type Audit

A full audit of all 14 tables identified four problem patterns:

| Pattern | Example | Fix |
|---------|---------|-----|
| Date stored as string | `races.date` | Task 2 |
| Duration stored as string | `lap_times.time` | Task 3 |
| Integer stored as float (nulls) | `results.grid` | Task 4 |
| String mislabelled as object | `results.positionText` | Low priority |

Full audit recorded in `Step2_Task1_DataType_Audit.md`.

---

## Task 2 — Casting Date Columns

### Columns Cast

| Table | Columns | Method |
|-------|---------|--------|
| races | date, fp1_date, fp2_date, fp3_date, quali_date, sprint_date | `pd.to_datetime(errors='coerce')` |
| drivers | dob | `pd.to_datetime(errors='coerce')` |

### Code

```python
date_cols = ['date', 'fp1_date', 'fp2_date', 'fp3_date',
             'quali_date', 'sprint_date']

for col in date_cols:
    dfs['races'][col] = pd.to_datetime(dfs['races'][col], errors='coerce')

dfs['drivers']['dob'] = pd.to_datetime(dfs['drivers']['dob'], errors='coerce')
```

### Why `errors='coerce'`

If a value cannot be parsed as a date, `coerce` converts it to `NaT`
(Not a Time) instead of crashing. This preserves our structural nulls.

### Verification Result

| Column | Type | Nulls | Reason for Nulls |
|--------|------|-------|------------------|
| races.date | datetime64 | 0 | Every race has a date |
| races.fp1_date | datetime64 | 1,035 | FP data from 2006 only |
| races.fp2_date | datetime64 | 1,053 | Structural — pre-2006 |
| races.fp3_date | datetime64 | 1,065 | Structural — pre-2006 |
| races.quali_date | datetime64 | 1,035 | Structural — pre-2006 |
| races.sprint_date | datetime64 | 1,141 | Sprints from 2021 only |
| drivers.dob | datetime64 | 0 | Every driver has a DOB |

All null counts match Step 1 predictions — structural nulls preserved.

---

## Task 3 — Parsing Time Strings to Milliseconds

### The Challenge

F1 time data appears in **four** different string formats across the
dataset. A single robust parser handles all of them.

| Format | Example | Meaning |
|--------|---------|---------|
| Seconds only | `"49.111"` | Pit stop duration |
| M:SS.mmm | `"1:27.452"` | Standard lap time |
| H:MM:SS.mmm | `"2:05:05.152"` | Long lap (red flag / safety car) |
| Null | `NaN` | Missing / structural |

### The Function

```python
def time_to_ms(time_str):
    if pd.isna(time_str):
        return np.nan

    parts = time_str.split(":")

    if len(parts) == 1:
        # plain seconds — e.g. "49.111"
        seconds = float(parts[0])
        total_ms = seconds * 1000

    elif len(parts) == 2:
        # M:SS.mmm
        minutes = float(parts[0])
        seconds = float(parts[1])
        total_ms = (minutes * 60000) + (seconds * 1000)

    elif len(parts) == 3:
        # H:MM:SS.mmm
        hours   = float(parts[0])
        minutes = float(parts[1])
        seconds = float(parts[2])
        total_ms = (hours * 3600000) + (minutes * 60000) + (seconds * 1000)

    return int(total_ms)
```

### Unit Conversion Logic

```
1 second  = 1,000 ms
1 minute  = 60,000 ms      (60 × 1000)
1 hour    = 3,600,000 ms   (60 × 60 × 1000)

total_ms = (hours × 3600000) + (minutes × 60000) + (seconds × 1000)
```

### Validation — The Key Quality Gate

`lap_times` already contains an official `milliseconds` column.
Parsing `lap_times.time` independently and comparing against it
proves the function is correct.

```python
mismatches = dfs['lap_times'][
    dfs['lap_times']['time_ms'] != dfs['lap_times']['milliseconds']
]
```

| Metric | Value |
|--------|-------|
| Total rows tested | 618,766 |
| Mismatches | **0** |
| Result | ✅ Parser validated against official data |

> The 3-part (hour) format was discovered *because* of this validation —
> 43 initial mismatches were all long laps in the `H:MM:SS.mmm` format.
> Adding that branch brought mismatches to zero.

### Columns Parsed

| Source Column | New Column | Nulls Preserved |
|---------------|-----------|-----------------|
| lap_times.time | time_ms | ✅ 0 → 0 |
| qualifying.q1 | q1_ms | ✅ 163 → 163 |
| qualifying.q2 | q2_ms | ✅ 4,787 → 4,787 |
| qualifying.q3 | q3_ms | ✅ 7,143 → 7,143 |
| results.fastestLapTime | fastestLapTime_ms | ✅ 18,535 → 18,535 |
| sprint_results.fastestLapTime | fastestLapTime_ms | ✅ 13 → 13 |
| pit_stops.duration | duration_ms | ✅ 3 → 3 |

> New `_ms` columns are added alongside the originals rather than
> replacing them — the original strings are kept for display/reference.

---

## Task 4 — Casting Float Columns to Nullable `Int64`

### The Concept

| Type | Holds Nulls? | Use |
|------|-------------|-----|
| `int64` (lowercase) | ❌ No | Pure integers, no missing values |
| `Int64` (capital) | ✅ Yes | Integers WITH possible nulls |

Pandas downcasts integer columns to `float64` whenever nulls are
present (e.g. `position` becomes `45.0` instead of `45`). The nullable
`Int64` type restores whole-number semantics while still allowing
`<NA>` for missing values.

### Detection Code

```python
# Find every float column that actually contains only whole numbers
for name, df in dfs.items():
    for col in df.select_dtypes(include='float64').columns:
        non_null = df[col].dropna()
        if len(non_null) > 0 and (non_null == non_null.astype(int)).all():
            print(f"{name}.{col}")
```

### Columns Cast (20 total)

| Table | Columns |
|-------|---------|
| constructor_standings | position |
| driver_standings | position |
| drivers | number |
| pit_stops | milliseconds, duration_ms |
| qualifying | q1_ms, q2_ms, q3_ms |
| results | number, grid, position, milliseconds, fastestLap, rank, fastestLapTime_ms |
| sprint_results | position, milliseconds, fastestLap, rank, fastestLapTime_ms |

### Cast Code

```python
int_cols = {
    'constructor_standings': ['position'],
    'driver_standings': ['position'],
    'drivers': ['number'],
    'pit_stops': ['milliseconds', 'duration_ms'],
    'qualifying': ['q1_ms', 'q2_ms', 'q3_ms'],
    'results': ['number', 'grid', 'position', 'milliseconds',
                'fastestLap', 'rank', 'fastestLapTime_ms'],
    'sprint_results': ['position', 'milliseconds', 'fastestLap',
                       'rank', 'fastestLapTime_ms'],
}

for table, cols in int_cols.items():
    for col in cols:
        dfs[table][col] = dfs[table][col].astype('Int64')
```

---

## Step 2 Outcome

| Achievement | Detail |
|-------------|--------|
| Date columns corrected | 7 columns → datetime64 |
| Time strings parsed | 7 columns → milliseconds |
| Parser accuracy | 0 mismatches / 618,766 rows |
| Integer columns restored | 20 columns → Int64 |
| Nulls preserved throughout | All structural & logical nulls intact |

---

## What's Next — Persistence + Step 3

Before Step 3, the cleaned in-memory dataframes are saved to **Parquet**
format to preserve dtypes and avoid re-running all transformations.

Step 3 then tackles the **null-handling strategy** — applying the
category rules from `Step1_Data_Quality_Decisions.md` to each remaining
null, and building the derived feature flags (`is_dnf`, `reached_q3`,
`has_permanent_number`).

---

*F1 Data Pipeline · Step 2 · Data Type Casting & Standardisation · 2026*