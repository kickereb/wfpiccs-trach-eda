# Research Protocol (draft v0.1)

## Title
Association between timing of tracheostomy and outcomes in critically ill children:
a multinational cohort study (Argentina — SATI-Q; Australia/New Zealand — ANZPICR)

## Background
Tracheostomy is used in a small but high-acuity subset of PICU admissions (~5% in the
SATI-Q cohort, see prior EDA) to facilitate prolonged ventilator weaning or manage upper
airway pathology. The optimal *timing* of tracheostomy relative to ICU admission /
intubation remains debated in paediatric critical care — earlier tracheostomy may reduce
sedation burden and ICU length of stay, but selection for early vs. late trach is
confounded by illness trajectory. This study uses two independent national/regional
registries to examine the association between tracheostomy timing and outcomes.

## Framing
This is not a question of whether tracheostomy is "good" or "bad" — both arms carry risk.
Staying intubated without a trach accrues risk with time (ventilator-associated
pneumonia, accidental extubation, sedation burden, other device-time-dependent harm).
Tracheostomy itself carries front-loaded procedural/early risk once performed. Timing is
the variable that trades one risk profile for the other, and the practical question is
where the crossover sits — i.e. whether there's a point past which the accumulated risk
of waiting exceeds the risk of proceeding, and whether that point is visible in the data.
Every outcome below should be read with this framing: not "is trach bad" but "which side
of the trade-off is worse, and when does that flip."

## Objective
Among children who received a tracheostomy during PICU admission, determine whether
timing of tracheostomy placement (days from ICU admission) is associated with:
- **Primary outcome:** ICU/hospital mortality
- **Secondary outcomes:** ICU length of stay, hospital length of stay, duration of
  mechanical ventilation, unplanned ICU readmission
- **"Cost of waiting" outcomes** (SATI-Q only, from `FiCompUti_2015_2025.csv` — device/
  immobility-time-dependent harms that accrue the longer a child remains intubated without
  a trach): ventilator-associated pneumonia (`NAR`), accidental extubation
  (`EXTUBACION_ACCIDENTAL`), CLABSI (`BACTAC`), CAUTI (`UTI`), pressure ulcers (`ESCARAS`).
  **Coding gotcha (corrected):** the data dictionary claims `0=Yes, 1=No` for these
  fields, but that is empirically wrong for this export — value `1` occurs in only
  1-2.3% of admissions across all five fields, matching real-world incidence, whereas
  treating `0` as "Yes" would mean ~98% of admissions had VAP. **Use standard coding:
  `1`=Yes/event happened, `0`=No.** Verified against raw value counts before trusting this
  over the dictionary text — don't re-invert. No equivalent complications file has been identified yet
  on the ANZPICR side (the diagnosis file's adverse-event codes may partially cover this —
  check the full ANZPICR diagnosis code list before assuming it's SATI-Q-only).

## Design
Retrospective, multinational, dual-registry cohort study. Each registry analysed
separately first (given differing case-mix, coding practice, and follow-up), then
pooled/compared via a harmonised variable set (two-stage: per-country estimates -> combined
interpretation, not a naive pooled regression, given between-registry heterogeneity).

## Data sources

| Registry | Country | File(s) used | Trach identification | Timing source |
|---|---|---|---|---|
| SATI-Q | Argentina | `FIVARAPA_2015_2025.csv` (+ `FiPIM3`/`FiPim` for severity) | `TRAQ`=1 | `TRAQFI` (placement date) − `FECHAING` (admission date) |
| ANZPICR | Australia/NZ | `ANZPICR_ADM.csv` + `ANZPICR_DIAG.csv` (+ `ANZPICR_EPI.csv` optional) | `ADX`=1506 ("PostOp - ENT Tracheostomy") | `ADX_DHr` (procedure datetime, mandatory when `ADX_CAT`=3) − ICU admission datetime (`ICUAdmYear/Month/Day/Hour`) |

**Key harmonisation decision:** ANZPICR has no dedicated tracheostomy field in the
admissions file — diagnosis code 1506 in `ANZPICR_DIAG` is the only marker, and it only
carries a timestamp (`ADX_DHr`) when `ADX_CAT`=3 (occurred during this ICU stay), which is
also the only case relevant to "timing since admission." Rows where the trach was
pre-existing (`ADX_CAT`=1) or already present on arrival must be excluded from the timing
analysis (analogous to excluding SATI-Q's `TRAQI`=1 "present on admission" patients) —
this cohort is about *new* tracheostomies during PICU care, not patients who arrived with
one.

## Cohort definition

**Include:** PICU admissions where a new tracheostomy was placed during that admission.
- SATI-Q: `TRAQ`=1 AND `TRAQI`=0 (used during stay, not present on admission), with a
  valid non-negative `TRAQFI` − `FECHAING`.
- ANZPICR: has an `ADX`=1506 record with `ADX_CAT`=3 and non-null `ADX_DHr`.

**Exclude:** missing/implausible dates (e.g. negative timing, the data-quality issue
already flagged in the initial EDA — some SATI-Q rows have `TRAQFF` before `TRAQFI`),
admissions with no recorded outcome.

## Exposure
**Days from ICU admission to tracheostomy placement**, analysed:
- Continuously (primary model)
- Dichotomised as *early* vs *late* for descriptive/sensitivity analysis — **cutoff not
  yet fixed**; adult literature commonly uses 7 or 10 days, but paediatric norms differ
  and this should be confirmed against paediatric trach literature or datathon
  supervisor guidance before finalising. Treat any cutoff used before that as provisional.

## Outcomes
- Primary: in-ICU or in-hospital mortality (`RESTRAT`/`RESULTADOEGRESOH` for SATI-Q;
  `DIED_IN_ICU`/`DIED_IN_HOSP` for ANZPICR)
- Secondary: `DIAS` / `ICU_HRS` (ICU LOS), hospital LOS, ventilation duration
  (`ARM`-linked dates for SATI-Q; `MECHVENT_HRS`/`INV_HRS` for ANZPICR), unplanned
  readmission (`REINGRESO` / `READMITTED`)

## Covariates / confounders (for adjustment)
Age, sex, admission category (elective vs emergency), primary diagnosis group,
severity of illness at admission (PIM/PIM3 score — `FiPim`/`FiPIM3` for SATI-Q,
`PIM3_LR/HR/VHR` or `PIM3RoD` for ANZPICR), invasive ventilation status, registry/country
(as a stratifying or random-effect variable, not pooled naively).

## Key methodological risk: immortal time / survivorship bias
Children who receive an **early** tracheostomy cannot, by definition, have died or been
discharged before it was placed — while children who go on to a **late** tracheostomy
have already "survived" the intervening period on the ventilator. Naively comparing
mortality between early- and late-trach groups will be biased toward making early
tracheostomy look protective. This must be addressed methodologically (e.g. landmark
analysis from a fixed post-admission day, or time-varying exposure / Cox modelling with
trach as a time-dependent covariate) rather than a simple two-group comparison — flagging
this now so the analysis notebook is built around a defensible design from the start,
not retrofitted later.

## Planned analysis stages
1. **Merge & extract** (this step) — build one harmonised, patient-level dataset per
   registry with: id, registry, age, sex, admission/discharge dates, severity score,
   trach flag, days-to-trach, LOS, ventilation duration, mortality outcome, readmission.
2. Descriptive comparison of trach cohort vs non-trach cohort, and across registries.
3. Time-to-event / landmark analysis of trach timing vs outcomes (per registry).
4. Cross-registry comparison / discussion (not pooled regression, given heterogeneity).

## Open questions to resolve with the team
- Final early/late cutoff (or stay fully continuous + landmark approach).
- Whether to restrict to a specific age band (SATI-Q pools neonates/paeds/adults in one
  file via `TIPO`; ANZPICR is already paediatric ≤18y).
- How to treat SATI-Q's known negative-duration date errors (already found in EDA).
