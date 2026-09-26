# Handoff — Claude (chat) <-> Claude Code (VSCode)

Shared working notes between the two Claude sessions. Claude (chat) does thinking/ideation,
Claude Code (VSCode) does the actual coding/running. Neither session can message the other
directly, so this file is the bridge — always read it before starting work, always update
your section when you hand off.

## To VSCode Claude Code

_(tasks queued up for VSCode to implement/run)_

- [ ] **Task 1 — Build `01_merge_extract.ipynb`** (new notebook, local only, runs in the
  `data_science` conda env). Read [RESEARCH_PROTOCOL.md](RESEARCH_PROTOCOL.md) first for
  the full rationale — this task implements its "Merge & extract" stage only.

  Goal: produce two harmonised, patient-level tables (one per registry — do NOT force
  them into one schema yet, just align column *names/meanings* where they overlap) saved
  locally as parquet (gitignored, do not commit): `derived/sati_q_cohort.parquet` and
  `derived/anzpicr_cohort.parquet`.

  **SATI-Q side:**
  1. Load `SATI-Q_WFPICCS_DATATHON_DATASET/FIVARAPA_2015_2025.csv` (master, one row per
     admission; composite key = `TIPODNI`+`DNI`+`FECING`).
  2. Left-join severity scores on that same composite key from
     `FiPIM3_2016_2025.csv` (preferred, newer score) and fall back to `FiPim.csv` where
     `FiPIM3` is missing (`SCOREPIM3`/`PROBPIM3` vs `SCOREPIM`/`PROBPIM`) — keep both
     source columns plus one coalesced `severity_score`/`severity_prob` pair.
  3. Parse dates (`dayfirst=True`): `FECHAING`, `FECEGR`, `TRAQFI`, `TRAQFF`, `FECINGH`,
     `FECEGRH`.
  4. Derive: `age`, `sex`=`SEXO`, `los_days`=`DIAS`, `mech_vent`=`ARM` flag,
     `mortality` (derive from `RESTRAT`/`RESULTADOEGRESOH` — inspect actual values first
     and document the mapping you use in the notebook, don't guess silently),
     `readmission`=`REINGRESO`.
  4a. Left-join `FiCompUti_2015_2025.csv` on the same composite key for the "cost of
     waiting" outcomes (see RESEARCH_PROTOCOL.md's Framing section): `vap`=`NAR`,
     `accidental_extubation`=`EXTUBACION_ACCIDENTAL`, `clabsi`=`BACTAC`, `cauti`=`UTI`,
     `pressure_ulcer`=`ESCARAS`. **These source fields are coded `0=Yes, 1=No` — invert
     them when deriving the boolean columns so `True` means the event happened**, and say
     so explicitly in a markdown cell (this is exactly the kind of silent-inversion bug
     that's easy to ship).
  5. Build the **timing cohort**: rows where `TRAQ`=1 AND `TRAQI`=0 (new trach, not
     present on admission) AND `TRAQFI`/`TRAQFF` both non-null AND
     `(TRAQFI - FECHAING).days >= 0` (drop the known negative-duration data errors found
     in the earlier EDA — count and report how many rows this drops).
     `days_to_trach` = `(TRAQFI − FECHAING).days`.
  6. Keep a `registry`="SATI-Q" column and a row for every trach patient (not just the
     clean-date subset) — add an `eligible_for_timing_analysis` boolean instead of
     silently dropping rows from the output table, so stage 2/3 can decide.

  **ANZPICR side:**
  1. Load `ANZPICR_WFPICCS_DATATHON_DATASET/ANZPICR_ADM.csv` (master; key = `DSiteID`+
     `ICU_NO`+`DURNo`). This file is 59MB — read only needed columns
     (`usecols=`) to keep memory sane: id fields, `SEX`, `AGE`, `ICUAdmYear/Month/Day/Hour`,
     `ICUDisYear/Month/Day/Hour`, `ICU_HRS`, `HospAdmYear...HospDisHour`, `OUTCOME`,
     `DIED_IN_ICU`, `DIED_IN_HOSP`, `READMITTED`, `ELECTIVE`, `PDX`, `PIM3_LR/HR/VHR`,
     `PIM3RoD`, `MECHVENT_HRS`, `INV_HRS`, `RS_HR124`.
  2. Load `ANZPICR_DIAG.csv`, filter to `ADX`==1506 (tracheostomy). Split into
     `ADX_CAT`==3 rows (new trach, has `ADX_DHr`) vs others (pre-existing/acute-on-admission
     — exclude from timing cohort, same logic as SATI-Q's `TRAQI`).
  3. Join the filtered trach rows onto ADM by the composite key (a patient could in
     theory have >1 row — dedupe by earliest `ADX_DHr` and note in the notebook if that
     happens).
  4. Construct ICU admission datetime from `ICUAdmYear/Month/Day/Hour` and parse
     `ADX_DHr`; `days_to_trach` = `(ADX_DHr − icu_admit_dt)` in days (can be fractional —
     `ADX_DHr` has hour precision, admission does too).
  5. Derive equivalent columns to the SATI-Q side with matching names:
     `registry`="ANZPICR", `age`, `sex`, `los_days` (from `ICU_HRS`/24), `mech_vent` (from
     `RS_HR124` or `INV_HRS`>0 — pick one, document choice), `mortality` (from
     `DIED_IN_ICU`/`DIED_IN_HOSP` — document mapping), `readmission`=`READMITTED`,
     `severity_score`/`severity_prob` (from PIM3 fields — document which).
  6. Same `eligible_for_timing_analysis` flag as SATI-Q.

  **Output:** print a short summary at the end of the notebook — row counts at each
  filter step (total admissions -> trach admissions -> new-trach -> eligible for timing),
  for both registries — and write it as the first entry under "From VSCode Claude Code"
  below when done, plus flag anything in the data that didn't match what
  RESEARCH_PROTOCOL.md assumed (e.g. unexpected `RESTRAT`/`OUTCOME` value sets).

- [ ] **Task 2 — Build `02_eda.ipynb`** (depends on Task 1's `derived/sati_q_cohort.parquet`
  and `derived/anzpicr_cohort.parquet` existing — wait for Task 1 to be marked done below
  before starting this one). Local only, `data_science` conda env. This is the full
  exploratory pass feeding the RQ in RESEARCH_PROTOCOL.md — thorough over fast; every
  section should print/plot, not just compute silently.

  **0. Data-quality gate (run first, stop and report if it fails)**
  - Load both derived parquet files. Assert the expected columns exist
    (`registry, age, sex, los_days, mech_vent, mortality, readmission, severity_score,
    severity_prob, days_to_trach, eligible_for_timing_analysis`, plus source id columns).
  - Missingness table (% null per column, per registry), rendered, not just printed to
    console.
  - Re-derive and print the exclusion funnel Task 1 reported (total admissions -> trach ->
    new-trach -> timing-eligible) for both registries side by side, so if the numbers
    drift from what Task 1 logged, that's caught immediately.
  - Range-check `days_to_trach` (no negatives should remain — if any do, stop, do not
    proceed to analysis, report it back under "From VSCode Claude Code" instead).

  **1. Cohort description ("Table 1")**
  - For the *timing-eligible* cohort, by registry: N, age (median/IQR), sex split,
    severity score (median/IQR + % missing), % on mech vent, LOS (median/IQR), mortality
    %, readmission %. One combined table, registries as columns.
  - Also pull in the raw admission tables (`FIVARAPA` for SATI-Q, `ANZPICR_ADM` for
    ANZPICR) to add a **denominator row**: what fraction of all PICU admissions this
    trach-timing cohort represents, per registry — reuse/extend the numbers from the
    original standalone EDA (`WFPICCS_Trach_EDA.ipynb`) rather than recomputing from
    scratch if that notebook's logic still applies.

  **2. Exposure: distribution of `days_to_trach`**
  - Histogram + ECDF, one panel per registry, shared x-axis scale for comparability.
  - Descriptive stats table (min/25/50/75/max/mean/sd) per registry.
  - Boxplot comparing the two registries directly (clearly label if a formal statistical
    test is or isn't appropriate given different populations/eras — don't just run a
    Mann-Whitney U silently without commenting on whether it's a meaningful comparison).
  - Sensitivity table: for cutoffs at 3, 5, 7, 10, 14, 21 days, show resulting group sizes
    (early vs late) per registry — this informs the "open question" in
    RESEARCH_PROTOCOL.md about which cutoff to use later, it doesn't answer it.
  - Flag any implausible outliers (e.g. `days_to_trach` > ICU LOS, which shouldn't be
    possible — cross-check `days_to_trach <= los_days` and report violations).

  **3. Outcome distributions**
  - Mortality %, LOS distribution, mech vent duration (where available), readmission % —
    each by registry, as both a table and a plot (bar for mortality/readmission,
    histogram/boxplot for LOS and vent duration).

  **4. Confounder relationships with the exposure**
  - `days_to_trach` vs severity score (scatter + correlation, per registry) — are sicker
    kids trached earlier or later?
  - `days_to_trach` vs age (scatter + correlation, per registry).
  - `days_to_trach` distribution by admission category (elective/emergency where
    available) and by primary diagnosis group where derivable (SATI-Q doesn't have a
    diagnosis-group field in FIVARAPA itself — check `FiMotingD`/`FiMotingP` for a joinable
    admission diagnosis if time allows; ANZPICR has `PDX`).

  **5. Exploratory (clearly-labelled-as-biased) timing vs outcome views**
  Read the "immortal time / survivorship bias" section of RESEARCH_PROTOCOL.md before
  writing this section, and put an explicit markdown warning above these plots that they
  are descriptive/exploratory only and NOT the study's answer to the RQ:
  - Mortality % by `days_to_trach` quantile bin (e.g. quintiles), per registry.
  - Scatter/LOESS of `days_to_trach` vs `los_days`, per registry.
  - A simple Kaplan-Meier-style descriptive curve (survival/discharge over ICU days) split
    by early/late at the 7-day cutoff, per registry, again labelled exploratory — full
    landmark/time-varying analysis is deferred to the modelling stage (stage 3 in the
    protocol), not this notebook.

  **6. Cross-registry comparability check**
  - Side-by-side (faceted) versions of the key plots above (severity distribution, age
    distribution, LOS distribution) so it's visually obvious how similar/different the two
    cohorts are before anyone considers pooling them.

  **7. Output**
  - Write a markdown summary cell at the end: bullet list of the concrete findings, and a
    separate bullet list of data-quality issues / open questions surfaced (in addition to
    the ones already known from RESEARCH_PROTOCOL.md).
  - Do not modify or overwrite the Task 1 parquet files from this notebook — read-only.

## From VSCode Claude Code

_(results, blockers, questions — written back after doing a task)_

- Nothing yet.

## Notes

- Data lives in `SATI-Q_WFPICCS_DATATHON_DATASET/` and `ANZPICR_WFPICCS_DATATHON_DATASET/` —
  gitignored, local only, never commit it.
- `WFPICCS_Trach_EDA.ipynb` is the working notebook (also mirrored to a Colab copy on Drive).
- Env: use the `data_science` conda env (`~/anaconda3/bin/conda install -y -n data_science -c conda-forge "numpy<2" pandas matplotlib seaborn`) — plain pip numpy/pandas versions clash on this Mac.
