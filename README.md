# 🏎️ F1 Intelligence Dashboard

> A full-stack Formula 1 data analysis project covering **every race, driver, and constructor from 1950 to 2026** — built from raw data through to advanced analytics.

This project goes beyond standard "who won the most championships" analysis. The goal is a clean, well-engineered data pipeline feeding novel analytics like teammate-based ELO ratings, championship "what-if" simulations, and statistical upset detection.

---

## 📊 Project Status

🚧 **In active development** — built step by step, documented as I go.

| Stage | Description | Status |
|-------|-------------|--------|
| Step 1 | Data Inventory & Schema Mapping | ✅ Complete |
| Step 2 | Data Type Casting & Standardisation | ✅ Complete |
| Step 3 | Null-Handling Strategy & Feature Engineering | 🔜 Next |
| Step 4 | Outlier Detection & Treatment | ⬜ Planned |
| Step 5 | Feature Engineering | ⬜ Planned |
| Step 6 | Relational Database Build | ⬜ Planned |
| Step 7 | Data Validation & Quality Checks | ⬜ Planned |
| Step 8 | Analysis-Ready Layer | ⬜ Planned |

---

## 🗂️ The Dataset

Formula 1 race data sourced from [Kaggle (jtrotman)](https://www.kaggle.com/datasets/jtrotman/formula-1-race-data).

| Metric | Value |
|--------|-------|
| Time span | 1950 – 2026 |
| Total tables | 14 |
| Total rows | 744,335 |
| Races | 1,171 |
| Drivers | 865 |
| Constructors | 214 |
| Circuits | 78 |
| Lap time records | 618,766 |

All 14 tables are related through a central `races` hub with **100% foreign key integrity** (zero orphaned records).

---

## 📁 Repository Structure

```
f1/
├── data_pipeline/          # Notebooks + step-by-step documentation
│   ├── data_inventory.ipynb
│   ├── Standardisation.ipynb
│   └── *.md                # Pipeline decision docs for each step
├── dataset/                # Raw source CSVs (immutable — never modified)
├── data_cleaned/           # Cleaned, type-cast data (pipeline output)
└── .gitignore
```

> **Design principle:** raw data in `dataset/` is never overwritten. The pipeline always runs from raw → cleaned, so results are fully reproducible.

---

## 🔧 What's Been Built So Far

### Step 1 — Data Inventory & Schema Mapping
- Full inventory of all 14 tables (shapes, dtypes, memory, nulls)
- Entity-Relationship Diagram mapping every table relationship
- Validated all foreign keys — 100% integrity
- Classified every null into **structural / logical / genuine gap** categories

### Step 2 — Data Type Casting & Standardisation
- Cast all date columns to `datetime64`
- Built a custom **time-string parser** converting `"1:27.452"` → milliseconds, handling four different time formats
- **Validated against 618,766 official lap records with zero mismatches**
- Cast 20 integer columns to nullable `Int64` to correctly handle missing values
- Preserved all structural and logical nulls throughout

---

## 🎯 The Vision

Once the pipeline is complete, the project will power:

- **Teammate ELO Ratings** — driver skill measured through head-to-head teammate battles
- **Championship "What-If" Simulator** — replay any season under any era's points system
- **Upset Detector** — the biggest statistical shocks in 75 years of racing
- **Constructor Fragility Map** — pace vs reliability across every team-season

---

## 🛠️ Tech Stack

- **Python** — pandas, numpy
- **Jupyter Notebooks** — exploratory analysis & pipeline
- **Planned:** DuckDB, scikit-learn, Plotly / D3.js, Streamlit

---

## 👤 Author

**Swetang Pandit**

- LinkedIn: [Swetang Pandit](https://www.linkedin.com/in/swetang-pandit/)
- GitHub: [@red-devil002](https://github.com/red-devil002)

---

## 📌 Note

This is a personal learning project built to deepen my skills in data engineering, analysis, and pipeline design — combining my passion for Formula 1 with hands-on data work. Documentation for each pipeline step lives in `data_pipeline/`.

---

*Built with a love for racing and clean data. 🏁*
