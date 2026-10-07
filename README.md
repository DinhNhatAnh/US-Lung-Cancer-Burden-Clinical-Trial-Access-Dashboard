# US Lung Cancer Burden & Clinical Trial Access Dashboard

An end-to-end analytics project: clean a messy public-health dataset, model it, and deliver a 2-page Power BI dashboard that tells leadership **where lung cancer burden is highest and whether clinical trial access keeps pace with it**.

![Page 1 - Cancer Burden](images/page1_cancer_burden.png)
![Page 2 - Trial Access](images/page2_trial_access.png)

---

## 1. Business Problem

A client asked for a state-level, monthly view of lung cancer in the United States (50 states + DC) to answer two questions:

1. **Burden:** How many new cases and deaths are there, how late are patients diagnosed, and which states and regions carry the most risk?
2. **Access:** Are clinical trials (and trial sites) available where the burden is highest?

The dashboard is designed for managers and executives: three KPIs per page, one clear story per page, and filters for Year, Month, Region and State.

## 2. Tools & Skills

| Area | Tools |
|---|---|
| Data cleaning & profiling | Python (pandas), Power Query |
| Data modeling | Power BI (galaxy schema / star schemas sharing dimensions) |
| Calculations | DAX (KPIs, MoM / YoY, ranking, tooltips) |
| Visualization | Power BI Desktop (Shape Map, Decomposition Tree, Scatter, Donut, Table) |

## 3. Dataset

Source file: `Lung_cancer_dataset.xlsx` (synthetic / educational data, per the dataset's own source note).

| Sheet | Grain | Rows | Description |
|---|---|---|---|
| `cancer_details` | State × Month | 1,836 (51 states × 36 months) | Population, new cases, deaths, stage breakdown, smoking / screening / survival rates, treatment centers |
| `Trail_details` | Trial × State × Snapshot month | 20,210 | Trial title, phase, status, sponsor type, intervention, subtype, active site count, target enrollment |
| `State_Mapping` | State | 51 | Standard state names, codes and regions |

Coverage: **January 2023 – December 2025**, 431 distinct trials.

## 4. Data Cleaning

The raw data was deliberately messy. Main issues and fixes:

| Issue | Example | Fix |
|---|---|---|
| Inconsistent state names | 102 spelling variants for 51 states (`ALABAMA`, `ca`, `MO`) | Standardized on `State_Code`, joined to `State_Mapping` |
| Numbers stored as text | `5,070,162 people`, `cases`, `sites` | Stripped non-numeric characters, cast to numeric |
| Mixed percentage formats | `17.5%`, `17.5`, `0.181` | Normalized everything to a 0–100 scale |
| Recruitment status | 10 variants (`open-recruiting`, `NYR`, `complete`) | Mapped to 4 values: Recruiting, Active Not Recruiting, Not Yet Recruiting, Completed |
| Phase | 15 variants (`PHASE1\|PHASE2`, `Phase I/II`, `Phase 1/Phase 2`) | Mapped to 5 values: Phase 1, Phase 1/2, Phase 2, Phase 2/3, Phase 3 |
| Casing of text fields | Titles, sponsors, interventions | Trim + proper case, then restore acronyms (AI, CT, KRAS, NSCLC, DNA) |

Integrity checks that passed: complete 51 × 36 state-month grid, no duplicate state-month keys, stage cases always sum to new cases, deaths never exceed new cases, each state belongs to exactly one region.

## 5. Data Model

```
Dim_Date  ──┬── Fact_Cancer   (State × Month)
            └── Fact_Trial    (Trial × State × Snapshot month)
Dim_State ──┬── Fact_Cancer
            └── Fact_Trial
Dim_Trial ──── Fact_Trial
```

- `Dim_State` (from `State_Mapping`) is the single source for state names, regions and map locations.
- The two fact tables are **never joined directly**; they share only the date and state dimensions.
- `Dim_Date` is a complete calendar marked as the date table.

## 6. KPI Definitions

| KPI | Definition |
|---|---|
| **New Cancer Cases** | Sum of new lung cancer cases in the selected period |
| **Avg Monthly Mortality / 100K** | Average of monthly rates: Deaths ÷ Population × 100,000 |
| **Late-Stage Diagnosis %** | (Regional + Distant cases) ÷ New cases. Unknown stage stays in the denominator |
| **Ongoing Trials** | Distinct trials whose status is not *Completed* at the latest selected month |
| **Recruiting Trials** | Distinct trials with status *Recruiting* (a subset of Ongoing) |
| **Active Sites per 1K Cases** | Active sites of ongoing trials ÷ new cases × 1,000 |

Time comparisons: **MoM** vs the previous month, **YoY** vs the same period last year. Late-stage change is best read in percentage points.

## 7. Dashboard Pages

### Page 1 - Cancer Burden
- **KPI cards:** New Cancer Cases, Avg Monthly Mortality/100K, Late-Stage Diagnosis % (each with MoM, YoY and a trend sparkline)
- **Shape Map:** mortality per 100K by state
- **Stage Distribution:** Localized / Regional / Distant / Unknown
- **Quarterly Cases vs Deaths**
- **Cancer Death Breakdown (Decomposition Tree):** Deaths → Smoking Prevalence band → Region → State

### Page 2 - Clinical Trial Access vs Burden
- **KPI cards:** Ongoing Trials, Recruiting Trials, Active Sites per 1K Cases
- **Burden vs Trial Access scatter:** mortality (X) vs active sites per 1K cases (Y), split into four quadrants; the bottom-right quadrant is **high burden, low access**
- **Priority States:** ranking of high-burden states with the lowest trial access
- **Trial detail table:** title, phase, sponsor type, status, intervention, active sites, target enrollment

## 8. Key Findings

**Burden**
- Across Jan 2023 – Dec 2025 the data records about **528K new cases** and **251,585 deaths**.
- Annual deaths decline steadily: roughly **89.2K (2023) → 83.8K (2024) → 78.5K (2025)**, a faster fall than the fall in new cases.
- **Late-stage diagnosis is stuck near 70%.** Distant-stage cases alone are **47.74%** of all cases, Regional 22.52%, Localized 25.25%, Unknown 4.5%.
- The highest mortality per 100K sits in the **Southern states**: Kentucky, West Virginia, Mississippi, Tennessee and Arkansas lead the ranking in the latest month.
- In absolute numbers the South accounts for the most deaths (70.9K in the Low-smoking band alone), led by **Texas (23.6K)** and **Florida (17.5K)**. Counts follow population, so mortality *rate* is the fairer way to compare states.
- Mortality per 100K rises with smoking prevalence across the state bands.

**Access**
- The states with the **lowest trial access among high-burden states** are **West Virginia, Arkansas, Alabama, Missouri, Ohio, Kentucky, Oklahoma and Mississippi**.
- The bottom-right scatter quadrant (high mortality, low access) is populated mostly by Southern and Appalachian states, which is where screening and trial investment would reach the most people.
- Wyoming has no trial records in any month, and Alaska has none in the latest month.

## 9. Recommendations

1. **Prioritize trial-site expansion** in West Virginia, Arkansas, Alabama and Kentucky, where burden is high and access is lowest.
2. **Shift focus from treatment capacity to earlier detection:** with about 70% of cases found at Regional or Distant stage, expanded screening in high-smoking states is the main lever.
3. **Track Unknown-stage cases** (4.5%) as a data-completeness indicator.
4. **Rank states by rate, not count,** in executive reporting to avoid population-driven conclusions.

## 10. Assumptions & Limitations

- The dataset is synthetic; findings illustrate the analytical approach, not real epidemiology.
- *Ongoing* is defined as every status except *Completed* (includes *Not Yet Recruiting*). This should be confirmed with the client.
- A trial can run in several states and may have different statuses across them in the same month, so trial counts use distinct trial IDs at the selected snapshot month.
- Smoking bands are set at state level with thresholds agreed for this project; changing the thresholds changes the Decomposition Tree split.
- Small states have small denominators, so per-1K ratios swing more from month to month.

## 11. Repository Structure

```
├── README.md
├── data/
│   ├── raw/Lung_cancer_dataset.xlsx
│   └── clean/                       # cleaned fact & dimension tables
├── notebooks/                       # profiling & cleaning (Python)
├── powerbi/
│   └── Lung_Cancer_Dashboard.pbix
├── dax/
│   └── Lung_Cancer_Dashboard_DAX.dax
├── docs/
│   └── Client_Problem_Statement.pptx
└── images/
    ├── page1_cancer_burden.png
    └── page2_trial_access.png
```

## 12. How to Reproduce

1. Open `Lung_Cancer_Dashboard.pbix` in Power BI Desktop (or load the cleaned tables from `data/clean/`).
2. Build the relationships described in section 5 and mark `Dim_Date` as the date table.
3. Create an empty `_Measures` table, then load `dax/Lung_Cancer_Dashboard_DAX.dax` through DAX query view (*Update model with changes*).
4. Set the Year / Month slicers to the latest month for the headline view.

## 13. Author

**[Your Name]** · Data Analyst (Marketing · Sales · Banking)
[LinkedIn](https://www.linkedin.com/in/your-profile) · [Email](mailto:your.email@example.com)
