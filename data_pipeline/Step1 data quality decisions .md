# F1 Data Pipeline — Step 1: Data Quality & Semantics Decisions

**Project:** F1 Intelligence Dashboard (1950–2026)
**Pipeline Stage:** 1 of 8
**Status:** ✅ Complete

> These decisions govern how every null, inconsistency, and ambiguity
> in the dataset is handled — from raw ingestion through to the final
> analysis layer. Every decision here has a reason. Nothing is removed
> blindly.

---

## The Core Principle

> *"Data cleaning is not about removing nulls.*
> *It is about understanding what the null represents."*

Each null in this dataset tells one of three stories.
Understanding which story it is determines every downstream decision —
feature engineering, model accuracy, dashboard consistency,
and cross-era comparisons from 1950 to 2026.

---

## Q1 — How Should Null Values Be Classified?

Null values are **not treated uniformly**. Each null is interpreted
based on its context and classified into one of three categories.

---

### 🟡 Category 1 — Structural Nulls (Expected)

Nulls that exist because the field is **not applicable** for certain
records due to rules, formats, or historical context.

| Column | Reason |
|--------|--------|
| `qualifying.q2`, `q3` | Not present in older qualifying formats |
| `drivers.number` | Permanent numbers introduced in 2014 only |
| `races.fp1_date` / `fp2_date` / `fp3_date` | FP sessions only recorded from 2006 |
| `races.sprint_date` | Sprint races introduced in 2021 |
| `results.fastestLapTime` | Only recorded from ~1996 onwards |
| `results.fastestLapSpeed` | Only recorded from ~2004 onwards |

**Decision:** Retain as `NULL`. Do not impute or alter.
Preserving these nulls maintains historical correctness.

---

### 🟠 Category 2 — Logical Nulls (Informative)

Nulls that represent a **meaningful real-world outcome**.
The absence of data is itself the data.

| Column | Real-World Meaning |
|--------|-------------------|
| `results.position` | Driver did not finish (DNF/DSQ/DNS) |
| `results.time` | Driver did not finish — no finishing time recorded |
| `results.milliseconds` | Same as above |
| `sprint_results.position` | Driver did not finish sprint |

**Decision:** Retain as `NULL` but convert into explicit features
where needed for analysis.

```python
# Create a boolean DNF flag from statusId
# statusId = 1 means "Finished" — everything else is a non-finish
df["is_dnf"] = df["statusId"] != 1
```

---

### 🔴 Category 3 — Genuine Data Gaps (Data Issues)

Nulls caused by **incomplete or missing data collection** —
records that should exist but don't.

| Column | Issue |
|--------|-------|
| `pit_stops.duration` | 3 rows missing stop duration — should be recorded |
| `pit_stops.milliseconds` | Same 3 rows |
| `constructor_standings.position` | 1 row missing position |
| `driver_standings.position` | 13 rows missing position |
| `results.grid` | 20 rows missing grid position |
| `results.number` | 6 rows missing car number |

**Decision:** Investigate each case individually.
Drop or impute depending on the specific use-case and volume.

---

### Final Rule

```
For every null encountered in this pipeline:
  1. Identify which category it belongs to
  2. Apply the category's handling rule
  3. Never drop a null without first classifying it
```

---

## Q2 — What to Do With `constructor_results.status`?

### Observation

```
Total rows in constructor_results : 12,898
Nulls in status column            : 12,881  (99.9%)
Non-null values                   :     17  ( 0.1%)
```

The column is effectively empty. The 17 non-null values carry
no analytical weight.

### Decision — Drop the Column

```python
dfs["constructor_results"].drop(columns=["status"], inplace=True)
```

### Reasoning

| Factor | Detail |
|--------|--------|
| Sparsity | 99.9% null — no usable signal |
| Redundancy | Race outcome already captured in `results.statusId` |
| Schema cleanliness | Removing it simplifies joins and reduces confusion |
| Information loss | Zero — nothing of value is removed |

---

## Q3 — Are `qualifying.q2` and `q3` Nulls a Problem?

### Observation

```
qualifying.q1  →    163 nulls  out of 11,036  ( 1.5%)
qualifying.q2  →  4,787 nulls  out of 11,036  (43.4%)
qualifying.q3  →  7,143 nulls  out of 11,036  (64.7%)
```

### Classification — Structural Nulls ✅

These nulls are **not a data quality problem**. They reflect two realities:

1. **Elimination format** — only the top 15 drivers reach Q2,
   only the top 10 reach Q3. Being null in q2 means the driver
   was eliminated in Q1. That is correct and expected.

2. **Historical format** — the current three-session qualifying
   format (Q1/Q2/Q3) was introduced in 2006. Pre-2006 qualifying
   used entirely different formats where q2 and q3 concepts
   did not exist.

### Decision — Retain as `NULL`

Do not impute. Do not drop. The null carries meaning.

### Optional Feature Engineering

```python
# Boolean flags for era-aware analysis
df["reached_q2"] = df["q2"].notna()
df["reached_q3"] = df["q3"].notna()

# These become useful features in:
# - Driver performance profiling
# - Qualifying pace models
# - Era segmentation
```

---

## Q4 — Difference Between `position` and `positionOrder` in Results

### Definitions

| Column | Meaning | Nulls |
|--------|---------|-------|
| `position` | Official finishing classification — null for any non-finisher | 10,953 (40.1%) |
| `positionOrder` | Always-populated numeric ranking — DNFs ranked after finishers | 0 (0%) |

### Key Insight

`position` is null for every driver who did not finish —
retirement, disqualification, did not start, accident.
`positionOrder` is **always set**, ranking finishers first
then non-finishers in order of laps completed.

```python
# Proof — check positionText where position is null
mask = df["position"].isnull()
print(df[mask]["positionText"].value_counts())
# Returns: R (Retired), D (Disqualified), W (Withdrawn), N (Not classified)
```

### Decision

```python
# For ALL ranking and analytical logic
df["positionOrder"]   # ← always use this

# For display and reporting only
df["position"]        # ← use only when showing official results
```

> ⚠️ Using `position` for analysis silently drops 40% of race entries.
> Every model, ranking, and aggregation in this project uses
> `positionOrder`.

---

## Q5 — What Do Nulls in `drivers.number` Indicate?

### Observation

```
Total drivers             :   865
drivers with number       :    63  ( 7.3%)
drivers WITHOUT number    :   802  (92.7%)
```

### Explanation

Permanent driver numbers were introduced by the FIA in **2014**.
Before that, numbers were **reassigned each season** based on
the previous year's constructor standings — drivers did not own
a number. A driver racing in 1975 genuinely had no permanent number.
The null is historically correct.

### Classification — Structural Null ✅

### Decision — Retain as `NULL`

```python
# Do not impute. Do not assign arbitrary numbers to historical drivers.

# Optional era segmentation feature
df["has_permanent_number"] = df["number"].notna()

# This flag is directly useful for:
# - Era segmentation (pre-2014 vs post-2014)
# - Filtering modern vs historical driver analyses
# - ELO model era weighting
```

### Insight

The 63 drivers with permanent numbers represent the **entire modern
grid** from 2014 onwards. The 802 without are the historical record —
every driver from Fangio to Schumacher. This boundary is one of the
cleanest era splits in the dataset.

---

## Decision Summary

| Question | Decision | Code Action |
|----------|----------|-------------|
| Q1 — Null classification | Three-category system | Classify before touching |
| Q2 — `constructor_results.status` | Drop the column | `.drop(columns=["status"])` |
| Q3 — `qualifying.q2` / `q3` nulls | Structural — retain | Add `reached_q2/q3` flags |
| Q4 — `position` vs `positionOrder` | Always use `positionOrder` | Replace all uses of `position` in logic |
| Q5 — `drivers.number` nulls | Structural — pre-2014 era | Add `has_permanent_number` flag |

---

## Impact on Downstream Pipeline

These decisions directly affect every subsequent step:

| Pipeline Step | Impact |
|---------------|--------|
| Step 2 — Type Casting | Know which nulls to preserve vs fill |
| Step 3 — Null Strategy | Category determines treatment rule |
| Step 5 — Feature Engineering | `is_dnf`, `reached_q3`, `has_permanent_number` flags |
| Step 6 — Database Build | Column drops applied at ingestion |
| ELO Model | Must use `positionOrder`, must handle DNFs correctly |
| Upset Detector | DNF classification affects pre-race probability targets |
| Dashboard | Era flags enable historical vs modern filtering |

---

*F1 Data Pipeline · Step 1 · Data Quality & Semantics · 2026*