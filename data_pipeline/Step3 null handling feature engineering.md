# F1 Data Pipeline — Step 3: Null Handling & Feature Engineering

**Project:** F1 Intelligence Dashboard (1950–2026)
**Pipeline Stage:** 3 of 8
**Notebook:** `Standardisation.ipynb`
**Status:** ✅ Complete

---

## What This Step Does

Step 1 *classified* nulls. Step 2 fixed *data types*. Step 3 **acts**:
it builds the analytical feature columns the project needs, and it
resolves every remaining null with an individual, evidence-based
decision.

| Part | Goal |
|------|------|
| Part A | Build derived feature columns |
| Part B | Investigate and resolve genuine data gaps |

---

## Part A — Feature Engineering

Six features were built, each validated before moving on.

### 1. `is_dnf` (results)

A boolean flag — did the driver fail to finish the race?

**The bug worth remembering:** the first attempt used
`statusId != 1` ("Finished"). This produced a **70.6% DNF rate** —
implausibly high. Investigation revealed that statuses like
`"+1 Lap"`, `"+2 Laps"` … `"+49 Laps"` are *classified finishers*
(drivers who completed the race a lap or more down), not retirements.

**The fix:** treat `"Finished"` AND all `"+N Laps"` statuses as finishes.

```python
finished_ids = dfs['status'][
    (dfs['status']['status'] == 'Finished') |
    (dfs['status']['status'].str.startswith('+'))
]['statusId'].tolist()

dfs['results']['is_dnf'] = ~dfs['results']['statusId'].isin(finished_ids)
```

**Result:** DNF rate corrected to **42.7%**.

**Validation — DNF rate by decade:**

| Decade | DNF Rate |
|--------|----------|
| 1950s | 51.0% |
| 1960s | 54.7% |
| 1970s | 53.2% |
| 1980s | 61.5% |
| 1990s | 54.0% |
| 2000s | 32.2% |
| 2010s | 19.7% |
| 2020s | 14.3% |

The steady decline from ~60% (unreliable early/turbo eras) to ~14%
(modern hybrid reliability) matches real F1 history — confirming the
feature is correct. The 1980s peak reflects the fragile turbo era.

### 2 & 3. `reached_q2` / `reached_q3` (qualifying)

Boolean flags for how far a driver progressed in qualifying.
A null `q2` time means the driver was eliminated in Q1.

```python
dfs['qualifying']['reached_q2'] = dfs['qualifying']['q2'].notna()
dfs['qualifying']['reached_q3'] = dfs['qualifying']['q3'].notna()
```

| Flag | Count | Total |
|------|-------|-------|
| reached_q2 | 6,249 | 11,036 |
| reached_q3 | 3,893 | 11,036 |

Counts shrink correctly at each stage (Q2 > Q3), matching the
elimination format.

> ⚠️ **Caveat:** `False` covers two different cases — a driver knocked
> out in Q1 (modern era) AND a driver from before 2006 when the
> Q1/Q2/Q3 format didn't exist. Era-specific qualifying analysis should
> filter to post-2006 races first.

### 4. `era` (races)

Decade label for cross-era analysis.

```python
def get_decade(year):
    return str((year // 10) * 10) + "s"

dfs['races']['era'] = dfs['races']['year'].apply(get_decade)
```

| Era | Races |
|-----|-------|
| 1950s | 84 |
| 1960s | 100 |
| 1970s | 144 |
| 1980s | 156 |
| 1990s | 162 |
| 2000s | 174 |
| 2010s | 198 |
| 2020s | 153 |

Race counts rise as the calendar expanded — 2010s peak, 2020s lower
(partial decade, 2020–2026).

### 5. `has_permanent_number` (drivers)

Flags modern-era drivers (permanent numbers introduced 2014).

```python
dfs['drivers']['has_permanent_number'] = dfs['drivers']['number'].notna()
```

63 True / 802 False — matches the Step 1 finding exactly.

### 6. `driver_fullname` (drivers)

Convenience column combining forename and surname.

```python
dfs['drivers']['driver_fullname'] = (
    dfs['drivers']['forename'] + " " + dfs['drivers']['surname']
)
```

Produces clean names like "Lewis Hamilton".

---

## Part B — Genuine Data Gap Investigation

Each remaining null was investigated by looking at the actual rows —
never by assumption. The recurring finding: almost every "genuine gap"
was in fact a **logical null in disguise**.

| Column | Nulls | Investigation Finding | Decision |
|--------|-------|----------------------|----------|
| results.grid | 20 | All from 2025 Qatar GP — grid data not yet entered (data lag). Drivers did race (positionText populated) | **Leave** — don't fabricate, don't drop a real race |
| results.number | 6 | All `positionText = "W"` (Withdrawn) — drivers entered but never raced | **Leave** — logical null, no car number is correct |
| driver_standings.position | 13 | All season openers (2025/2026 round 1), 0 points, `positionText = "-"` — no standings position exists before first result | **Leave** — position genuinely undefined |
| constructor_standings.position | 1 | Same cause — season opener | **Leave** |
| pit_stops.duration | 3 | 1994 Hungarian GP — rare pre-2011 pit records with duration never captured | **Leave** — unknowable, can't impute |

### The Core Lesson

> Not a single row needed dropping. Investigating before acting
> prevented silent data corruption. Dropping on instinct would have
> cost a Grand Prix (Qatar 2025), 6 real entries, 14 standings
> snapshots, and 3 pit stops.

**Decision principle applied throughout:**
- Never impute a value we cannot truly know (no fabrication)
- Never drop rows that represent real events (no data loss)
- A null with a clear meaning is more honest than a guessed value

---

## Features Summary

| Feature | Table | Type | Purpose |
|---------|-------|------|---------|
| is_dnf | results | bool | Did driver fail to finish |
| reached_q2 | qualifying | bool | Reached Q2 |
| reached_q3 | qualifying | bool | Reached Q3 |
| era | races | string | Decade label |
| has_permanent_number | drivers | bool | Modern vs historical driver |
| driver_fullname | drivers | string | Combined display name |

---

## Downstream Impact

| Consumer | Uses |
|----------|------|
| ELO model | is_dnf (who actually raced) |
| Era comparisons | era |
| Qualifying analysis | reached_q2 / reached_q3 |
| Upset detector | is_dnf |
| Dashboard filters | era, has_permanent_number |

---

## What's Next — Step 4

Step 4 covers **Outlier Detection & Treatment** — finding anomalous
values in the numeric columns (lap times, pit durations, points) and
deciding which are genuine extremes versus data errors.

---

*F1 Data Pipeline · Step 3 · Null Handling & Feature Engineering · 2026*