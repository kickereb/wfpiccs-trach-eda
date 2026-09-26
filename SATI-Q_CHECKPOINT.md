# SATI-Q (Argentina) Dataset — Checkpoint

Everything established about the Argentinian registry so far: what each file contains,
every question asked and how it was answered, findings, data-quality issues, and what's
still unexplored. Companion to [RESEARCH_PROTOCOL.md](RESEARCH_PROTOCOL.md) (the study
design) — this doc is the SATI-Q reference material behind it. ANZPICR is out of scope
here; see RESEARCH_PROTOCOL.md for the cross-registry picture.

---

## 1. What SATI-Q is

SATI-Q is Argentina's national paediatric intensive care quality-indicator program
(Sociedad Argentina de Terapia Intensiva). The datathon extract covers **PICU admissions
from 1 January 2015 to 31 December 2025**, across Argentine PICUs participating in the
program. Source: `WFPICCS Datathon Data Dictionaries/SATI-Q_Data Dictionary.xlsx` (the
`READ` sheet) and `SATI-Q_Data Dictionary.xlsx` per-table tabs.

**Composite primary key across every table:** `TIPODNI` + `DNI` + `FECING` (this triple
identifies one PICU admission and is how all ten files link together — there is no
single-column patient ID).

---

## 2. Files — what's in each, and how we confirmed it

Found via `ls`/`find` on `SATI-Q_WFPICCS_DATATHON_DATASET/`, row counts via `wc -l`,
headers via `head -1`, field meanings from the matching `Tabla_*` sheet in
`SATI-Q_Data Dictionary.xlsx` (read via `openpyxl`).

| File | Rows | Grain | Purpose |
|---|---|---|---|
| `FIVARAPA_2015_2025.csv` | 87,770 | 1 row / admission | **Master table** — demographics, LOS, device-use flags (incl. tracheostomy), outcome |
| `FiPIM3_2016_2025.csv` | 79,727 | 1 row / admission | PIM3 severity score (2016-2025, newer/preferred) |
| `FiPim.csv` | 32,315 | 1 row / admission | PIM2 severity score (2015-2019 only, fallback) |
| `FiCompUti_2015_2025.csv` | 88,865 | 1 row / admission | Device/immobility-associated complications (VAP, CLABSI, CAUTI, pressure ulcer, accidental extubation, + 3 more) |
| `FiPracticaARM_2015_2025.csv` | 69,386 | 1 row / **ventilation episode** | Mechanical ventilation episodes (start/end, device, complications, stop reason) — multiple rows per admission possible |
| `FiPracticaCVC_2015_2025.csv` | 61,511 | 1 row / catheter | Central venous/arterial line episodes |
| `FiPracticaFoley_2015_2025.csv` | 45,281 | 1 row / catheter | Urinary catheter episodes |
| `FiPracticaSNG_2015_2025.csv` | 40,984 | 1 row / tube | Nasogastric tube episodes |
| `FiMotingD_2015_2025.csv` | 181,261 | many rows / admission | All diagnoses recorded at admission (free text, Spanish) |
| `FiMotingP_2015_2025.csv` | 80,508 | 1 row / admission | Single standardised diagnostic **category** per admission |

---

## 3. Field reference (decoded)

### 3.1 `FIVARAPA` (master table) — 30 fields
`TIPODNI, DNI, FECING` (key) · `FECHAING` admission date · `REINGRESO` unplanned
readmission <48h (0/1) · `HC` de-identified MRN · `TIPO` patient type: **0**=paediatric
(1-191 months), **1**=adult (≥16y), **2**=neonate (<30 days) · `PACIENTE` de-identified
name · `FECEGR` PICU discharge date · `DIAS` PICU LOS in days · `EDAD` age, **unit
depends on `TIPO`**: months if 0, years if 1, days if 2 — never mix across `TIPO` without
converting · `SEXO` M/F · `PROCEDENCIA` source of admission, see §3.5 · `TRAQ`/`TRAQI`/
`TRAQE` tracheostomy used-during-stay / present-on-admission / present-on-discharge (S/N)
· `TRAQFI`/`TRAQFF` trach placement/removal dates · `SNG`/`SNGI`/`SNGE` nasogastric tube
(same pattern) · `ARM` invasive mechanical ventilation used during stay (S/N) · `SF`/
`SFI`/`SFE` urinary catheter (same pattern) · `FECINGH`/`FECEGRH` **hospital** (not
PICU) admission/discharge dates · `RESTRAT` ICU discharge outcome code, see §3.4 ·
`RESULTADOEGRESOH` hospital discharge outcome, free text (see §5 — messy) ·
`Dependencia` institution type: Social Security / Private / Public.

### 3.2 `FiCompUti` (complications)
`NAR`=VAP · `UTI`=CAUTI (catheter-associated UTI) · `BACTAC`=CLABSI (central-line
bloodstream infection) · `ESCARAS`=pressure ulcer · `EXTUBACION_ACCIDENTAL`=unplanned
extubation · plus `DESPLAZAMIENTO_SNG` (NG tube displacement), `CAIDA` (fall),
`INFECCION_HERIDA` (wound infection) — not yet used in analysis. Each has a matching
`Num*` column (episode count). **Coding: the dictionary states `0=Yes, 1=No` — this is
wrong, see §5.1.**

### 3.3 `FiPIM3` / `FiPim` (severity scores)
Both compute a paediatric mortality-risk score from first-hour-of-admission physiology
(pupil reaction, BP, base excess, FiO2/PaO2, ventilation status, elective/bypass/recovery
flags, diagnosis risk category). `SCOREPIM3`/`PROBPIM3` (or `SCOREPIM`/`PROBPIM` for the
older PIM2 version) are the derived score and predicted mortality probability — these are
what we use as `severity_score`/`severity_prob`. `FiPIM3` covers 2016-2025 and is
preferred; `FiPim` (PIM2, 2015-2019 only) is the fallback for earlier admissions.
`RISK`/`HIGHRISK`/`LOWRISK` encode specific high/low-risk admission diagnoses (e.g. code
301=cardiac arrest before PICU, 101=asthma) — not yet used.

### 3.4 `RESTRAT` — ICU discharge outcome (from `Restrat` reference tab)
| Code | Meaning |
|---|---|
| 0 | Missing |
| 1 | Discharge to inpatient ward |
| 2 | Discharge home |
| 3 | Discharge against medical advice |
| 4 | Transfer to another institution |
| **5** | **Death** |
| 6 | Intermediate care unit (IMCU) |
| 7 | Coronary care unit (CCU) |
| 8 | Other |
| 9 | Home care |
| 10 | Transfer to another ICU |
| 11 | Chronic care facility |

We use `RESTRAT==5` as `icu_mortality`. (Raw data has a stray code `12` appearing once —
not in the dictionary, treat as noise/possible data entry error, negligible n.)

### 3.5 `PROCEDENCIA` — source of admission
Emergency Department (1) · Medical ward (2) · Delivery room (3) · Ward, other hospital
(4) · ICU, other hospital (5) · IMCU (6) · Surgical ward (7) · Public area/scene (8) ·
CCU (9) · Elective OR (10) · Emergency OR (11) · Other (12) · Home care (13) · ED, other
hospital (14). 0 = missing. Not yet used in analysis.

### 3.6 `FiPracticaARM` (ventilation episodes) — richer than `FIVARAPA.ARM`
This is episode-level (start/end date per ventilation spell), unlike `FIVARAPA`'s single
stay-level `ARM` flag. `TIPO` distinguishes IMV (invasive)/VNI (non-invasive)/CAFO
(high-flow). `ARMUTIL` (ventilator model, ~90 codes, mostly irrelevant to analysis).
**`ARMCOMPL` (ventilation complications) independently includes code 5 = "Ventilator-
associated pneumonia (VAP)" and code 13 = "Unplanned/accidental extubation"** — i.e. this
file has its own VAP/accidental-extubation signal, separate from `FiCompUti`'s `NAR`/
`EXTUBACION_ACCIDENTAL`. **Not yet cross-checked against `FiCompUti` — a good validation
step for the complications analysis, not yet done.** `MOTIVO` (why ventilation
ended) distinguishes planned-successful (F1/F5) vs unplanned extubation (F2) vs death
(F3) — another possible complication signal, unused so far.

### 3.7 `FiPracticaCVC` — `MOTIVO` (removal reason) codes
Entry-site suppuration (1) · routine replacement (2) · end of treatment (3) · fever (5) ·
lab-confirmed infection (6) · accidental removal (7) · discharge (8) · death (9) ·
occlusion (10) · malposition/breakage (11) · suspected catheter infection (13) ·
mechanical complication (14) · discharged with CVC in place (15) · other (99). Not yet
used.

### 3.8 `FiMotingP` — standardised diagnostic category
Single category per admission, stored as **plain Spanish text, not a numeric code** (no
lookup needed): `Respiratorio` (29,987), `Postquirúrgico` (20,135), `Otros` (13,298),
`Neurológico` (8,142), `Causa Externa` (5,626 — "External cause"/injury),
`Cardiológico` (3,320). Confirmed by direct `awk` frequency count against the raw CSV,
not just the dictionary's category list.

### 3.9 `FiMotingD` — all admission diagnoses (free text)
Multiple rows per admission allowed. `DIAGING` is **plain Spanish free text**, not a
numeric code either (checked the same way as `FiMotingP`) — top values: "Insuficiencia
respiratoria" (respiratory failure, 11,895), "Neumonia o neumonitis" (10,135),
"Bronquiolitis" (8,570), "Convulsiones" (seizures, 6,493), various "Cronico ..." chronic
condition labels, "Shock septico", etc. `AD` marks each entry as Primary (P) / Current
(D) / pre-existing Antecedent (A) diagnosis.

---

## 4. Q&A log

**Q: Which SATI-Q files say anything about tracheostomy?**
A: `FIVARAPA` directly (`TRAQ`/`TRAQI`/`TRAQE`/`TRAQFI`/`TRAQFF`) — this is the primary
and really only source. Found by grepping every file's header row for trach-related
column names.

**Q: What fraction of admissions involve a tracheostomy, and how many are new
placements vs. already present on arrival?**
A: 5.03% of admissions (4,413 / 87,770) have `TRAQ`=1. Of those, `TRAQI`=1 (present on
admission) for 2,392, and `TRAQE`=1 (present on discharge) for 2,859. New placements
during the stay (`TRAQ`=1 & `TRAQI`=0) = 2,021. Found via `pandas.value_counts()` on the
raw flags.

**Q: How do trach and non-trach admissions compare on LOS, ventilation, and outcome?**
A: Trach admissions: median LOS 16 days vs. 4 for non-trach (mean 37 vs 8); 86.5% on
invasive ventilation vs. 52.6%. Found via `groupby(TRAQ_flag).describe()`/`.mean()`.

**Q: Is the tracheostomy rate changing over the 10-year period?**
A: No clear trend — bounces between 4.0% and 6.1% every year, 2015-2025 (yearly
groupby on admission year parsed from `FECHAING`).

**Q: What does `RESTRAT` actually mean, and which code is death?**
A: Read the `Restrat` reference tab in the dictionary — code 5 = Death (full table in
§3.4 above). Confirmed the raw data's value range (0-11, plus one stray `12`) matches.

**Q: What does `RESULTADOEGRESOH` (hospital-level outcome) contain?**
A: Free Spanish text, mostly blank (66,840 / 87,770 rows empty — only filled when
hospital-level follow-up was tracked). "Fallece" (dies) appears 4,640 times; "Alta
domiciliaria" (discharged home) 11,747 times; plus several messy/junk values (a stray
year "2020", a code "12", clinical notes like "Curado y epitelizado 100%") — this field
needs light cleaning before use, unlike `RESTRAT`. Found via `awk -F',' | sort | uniq -c`
on the raw column.

**Q: How is admission severity captured, and which score should be used?**
A: `FiPIM3` (PIM3, 2016-2025) is preferred; `FiPim` (PIM2, 2015-2019 only) is the
fallback for earlier admissions not covered by PIM3. Matched 78,710 admissions to PIM3
and used PIM2 to cover another 32,200 (with overlap — combined via `combine_first`).
Confirmed date ranges from each table's dictionary intro text.

**Q: Are the `FiCompUti` complication flags coded the way the dictionary says
(`0=Yes, 1=No`)?**
A: **No — the dictionary is wrong.** Checked raw `value_counts()` for all five fields:
value `1` occurs in only 1.1-2.3% of admissions across `NAR`/`UTI`/`BACTAC`/`ESCARAS`/
`EXTUBACION_ACCIDENTAL` — exactly the incidence you'd expect for real adverse events.
Treating `0` as "Yes" would mean ~98% of every PICU admission had VAP, which isn't
plausible. **Standard coding (`1`=Yes/event happened, `0`=No) is what's actually in the
data.** This was caught by cross-checking the dictionary's claim against empirical
frequency before trusting it — worth doing for every coded field, not just this one.
Full detail and fix history in RESEARCH_PROTOCOL.md.

**Q: Among children who got a *new* tracheostomy, what's the distribution of timing
(days from admission to placement), after cleaning bad dates?**
A: Of 2,021 new-trach admissions: 33 had `TRAQFF` before `TRAQFI` (impossible negative
duration — excluded) and a further 8 had a trach date beyond the recorded LOS
(`DIAS`) — also impossible, excluded. Remaining 1,899 eligible: median 19 days,
IQR 9-30, max 427 (a genuine long-stay outlier, not a data error — checked its `los_days`
was long enough to contain it). Built via a full merge pipeline joining `FIVARAPA` +
`FiPIM3`/`FiPim` + `FiCompUti` on the composite key, executed and saved to
`derived/sati_q_cohort.parquet` (gitignored).

**Q: Do complication rates (VAP, accidental extubation, CLABSI, CAUTI, pressure ulcer)
rise the longer a child waits for a tracheostomy?**
A: Yes, and strongly — monotonically across every one of the five complications. "Any
complication" rate: 13.6% in the earliest-trach group (≤6 days) up to 56.8% in the
latest (>34 days). VAP alone: 6.3% → 30.2%. Computed via quintile-binning
`days_to_trach` and taking the mean of each boolean complication flag per bin.

**Q: Is that rise just because sicker kids get delayed trachs (confounding by
indication), rather than the delay itself doing harm?**
A: Admission-time severity (`severity_prob`, from PIM3/PIM2) is only weakly correlated
with `days_to_trach` (r≈0.04); age correlates even more weakly (r≈-0.09). Neither is
strong enough to explain a 13.6%→56.8% jump in complication rate on its own — but
admission severity doesn't capture how the child's condition evolved *during* the stay,
so this rules out one specific confounder, not confounding in general.

**Q: What's a comparable "cost of waiting" signal for the other registry (ANZPICR),
which has no complications file?**
A: `ANZPICR_DIAG` code 465 ("Pneumonia or Pneumonitis") filtered to `ADX_CAT`=3 (acquired
during the ICU stay) is a usable proxy — broader than VAP specifically, but checked
against ANZPICR's own new-trach cohort: 34/175 (19.4%) carry it, close to SATI-Q's actual
VAP rate of 18.2% in its equivalent cohort. (This is an ANZPICR finding, logged here
because it was benchmarked directly against the SATI-Q VAP number above — full detail in
RESEARCH_PROTOCOL.md.)

**Q: In plain language, what does all this mean?** (asked directly, answered without
jargon)
A:
- About 5 in 100 kids admitted to these Argentine PICUs end up needing a tracheostomy at
  some point during their stay.
- Most tracheostomies are new, not pre-existing — roughly 2,000 of the ~4,400 trach
  admissions arrived without one and got it during the stay.
- Kids with a tracheostomy stay far longer in intensive care — typically 16 days versus
  4 days for everyone else.
- Almost 9 in 10 tracheostomy patients were on a breathing machine, versus about half of
  everyone else.
- The tracheostomy rate hasn't trended up or down over 10 years — it bounces between 4%
  and 6% of admissions every year.
- The longer a child waits for the tracheostomy, the more likely they are to pick up a
  device-related complication (infection, pneumonia, an accidental tube dislodgement) —
  rising from about 1 in 7 kids in the earliest-trach group to over half in the
  latest-trach group.

---

## 5. Data-quality issues found (with how they were caught and fixed)

### 5.1 `FiCompUti` 0/1 coding is backwards from the dictionary
See Q&A above. **Fix:** use `1`=Yes, `0`=No, ignore the dictionary text for this specific
field. This was wrong in the first draft of `RESEARCH_PROTOCOL.md`/`HANDOFF.md` and has
since been corrected in both.

### 5.2 Negative tracheostomy duration
33 of 2,021 new-trach admissions have `TRAQFF` (removal date) before `TRAQFI` (placement
date) — a data-entry error, not a real negative duration. **Fix:** excluded from the
timing-eligible cohort.

### 5.3 Tracheostomy date beyond recorded length of stay
A further 8 admissions have `TRAQFI - FECHAING` (days to trach) *greater* than the
recorded `DIAS` (total LOS) — i.e. the trach is dated after the patient was supposedly
already discharged. Includes one extreme outlier (~90 *years*, almost certainly a
mistyped year). **Fix:** excluded from the timing-eligible cohort. This check wasn't in
the original EDA pass — only surfaced when building the merged cohort and cross-checking
`days_to_trach` against `los_days` directly.

### 5.4 `RESULTADOEGRESOH` free-text messiness
Mostly blank (76% of rows), and among the filled values there are a handful of clearly
wrong entries (a bare year, a bare number, clinical progress notes instead of an
outcome). Not yet cleaned — currently using `RESTRAT` as the primary mortality field
instead, since it's far more complete (99.5% non-null) and cleanly coded.

---

## 6. Not yet done / open items on the SATI-Q side

- `FiPracticaARM`'s own VAP (code 5) and accidental-extubation (code 13) complication
  codes have not been cross-checked against `FiCompUti`'s `NAR`/`EXTUBACION_ACCIDENTAL` —
  worth doing as a validation of the complications analysis (do the two sources agree on
  which admissions had VAP?).
- `FiMotingD`/`FiMotingP` (diagnosis text/category) has not been joined into the trach
  timing cohort yet — would let the "cost of waiting" analysis be broken down by primary
  diagnosis group (e.g. is the complication-rate rise steeper for respiratory admissions
  than for post-surgical ones?).
- `FiPracticaCVC`/`FiPracticaFoley`/`FiPracticaSNG` (episode-level device data) not yet
  used at all — could give exact device-days exposure instead of the coarser stay-level
  `FiCompUti` flags.
- `RESULTADOEGRESOH` not cleaned/used as a secondary (hospital-level, vs. ICU-level)
  mortality outcome.
- PIM3 `RISK` categories (diagnosis-based risk tiers) not yet used as a covariate.
