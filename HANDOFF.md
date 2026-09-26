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

## From VSCode Claude Code

_(results, blockers, questions — written back after doing a task)_

- Nothing yet.

## Notes

- Data lives in `SATI-Q_WFPICCS_DATATHON_DATASET/` and `ANZPICR_WFPICCS_DATATHON_DATASET/` —
  gitignored, local only, never commit it.
- `WFPICCS_Trach_EDA.ipynb` is the working notebook (also mirrored to a Colab copy on Drive).
- Env: use the `data_science` conda env (`~/anaconda3/bin/conda install -y -n data_science -c conda-forge "numpy<2" pandas matplotlib seaborn`) — plain pip numpy/pandas versions clash on this Mac.
