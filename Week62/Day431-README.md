# CMS156v13 – "Use of High-Risk Medications in Older Adults" MACRA Display Discrepancy

## Overview

This document records the investigation into a reported defect in GlaceEMR's MACRA/QPP Flowsheet, where the **"Use of High-Risk Medications in the Elderly"** quality measure (CMS156v13, internal `measure_id = 238`) displayed **MET / NOT MET** statuses that appeared to contradict the raw values stored in the `quality_measures_patient_entries` table for a specific patient.

The investigation traced the issue through:
1. Clinical/prescribing data (`doc_presc`, `current_medication`)
2. The eCQM calculation logic (CMS156v13 CQL specification)
3. The MACRA UI rendering layer (`MacraFlowsheetDesktopView.java`)

**Conclusion:** The apparent UI/DB mismatch is **not a defect** — it is the intended behavior of an "inverse measure" flip implemented in the GWT client. The **actual open issue** is a suspected discrepancy between the CQL specification's dosage threshold logic and the calculated numerator value stored in `quality_measures_patient_entries`, which requires further backend investigation.

---

## Background / Context

- **Product:** GlaceEMR (Glenwood Systems) — multi-tenant healthcare EMR platform.
- **Module:** MACRA/QPP Flowsheet — displays Quality Measures (ECQM), EP (Promoting Interoperability) Measures, and PQRS measures per patient, per reporting year, for MIPS reporting.
- **Measure in scope:**
  - **CMS ID:** CMS156v13
  - **Internal Measure ID:** 238
  - **Title:** Use of High-Risk Medications in Older Adults
  - **Measure Steward:** NCQA (developed under CMS contract)
  - **Measure Type:** Process measure, Proportion scoring
  - **Improvement Notation:** **Lower score = better quality** (this is an "inverse" or "negative" measure — a MET numerator represents an undesirable clinical event, not a desirable one)
  - **Measurement Period (case data):** Jan 1, 2026 – Dec 31, 2026

**Why this matters:** Quality measure MET/NOT MET status feeds MIPS reporting and is visible to providers directly in the patient chart. Incorrect-looking statuses cause providers to question data integrity and can affect trust in the reporting pipeline, even when (as found here) the underlying calculation logic is partially correct by design.

---

## Problem Statement

**Reported by:** Provider (Dr. George Maly, M.D.) via support ticket, escalated by Gobala Krishnan (support engineer) to Althaf (senior/team lead).

**Symptom:**
> "I prescribed a high-risk medication for the patient in 2026, but the MACRA tab shows **NOT MET** for 'Use of High-Risk Medications in the Elderly.'"

**Affected patient/context:**
- Account #: `008516`
- `patientId`: `8516`
- `chartId`: `15608`
- Reporting Year: `2026`
- Tenant/DB: `malynew`
- Provider: George Maly, M.D. (`providerId = 2`)

**Observed UI state (MACRA Flowsheet):**

| Measure Row | UI Status |
|---|---|
| Use of High-Risk Medications in the Elderly (Criteria-1) | **NOT MET** |
| Use of High-Risk Medications in the Elderly (Criteria-2) | **MET** |
| Use of High-Risk Medications in the Elderly (Criteria-3) | **NOT MET** |

**Observed DB state (`quality_measures_patient_entries`, `measure_id = 238`, `reporting_year = 2026`, `patient_id = 8516`):**

| Criteria | Numerator | Denominator | Exclusion | Exception |
|---|---|---|---|---|
| 1 | 1 | 1 | 0 | 0 |
| 2 | 0 | 1 | 0 | 0 |
| 3 | 1 | 1 | 0 | 0 |

At first inspection, this looked like a **direct inversion**: every DB numerator value appeared opposite to the UI-displayed status, and Criteria-2 (MET) combined with Criteria-3 (NOT MET) initially looked **logically impossible** per the CQL specification (`Numerator3 = Numerator2 OR (Numerator1 AND NOT Numerator2)`, which reduces to `Numerator1 OR Numerator2` — if Numerator2 is true, Numerator3 must be true).

---

## Impact

- **Provider-facing confusion:** Clinicians may believe the EMR is failing to capture prescriptions correctly for MIPS/QPP reporting.
- **Potential trust/perception risk:** If unexplained, this could be perceived as a data-loss or calculation-engine defect, prompting unnecessary escalations across other patients/measures.
- **No confirmed data-loss or scoring defect** was found in the *display* layer — but a **potential real defect in the calc engine's dosage-threshold evaluation** was identified as a side-finding and requires separate resolution (see [Root Cause Analysis](#root-cause-analysis) and [Pending Work](#pending-work)).

---

## Environment Details

| Item | Value |
|---|---|
| Tenant / DB schema | `malynew` |
| Patient | Account# 008516 / patientId 8516 / William J. Orwig / DOB 1937-08-26 |
| Chart ID | 15608 |
| Encounter(s) referenced | 438717 (2025-08-15), 446985 (2026-02-09, refill-only), 450616 (2026-04-30) |
| Reporting Year | 2026 |
| Measure ID | 238 |
| CMS ID | CMS156 (v13.2.000) |
| Service Doctor | George Maly, M.D. (providerId 2) |
| Access method | Glenwood Systems Remote Support session, GlaceEMR desktop (GWT client), direct `psql` access to `malynew` |
| Backend stack (per prior context) | JSP/Struts legacy frontend (`glacelegacy_master`) + Spring Boot backend (`glaceemr_backend_new`) + PostgreSQL multi-tenant DB |
| MACRA client | GWT-based (`glaceemr_ui_master_1`), served via `glace-gwt.glaceemr.com/sdesktop/glaceemr.html` |

---

## System Components Involved

### Database Tables
- `quality_measures_patient_entries` — stores calculated IPP/Denominator/Numerator/Exclusion/Exception per patient, per measure, per criteria, per reporting year (source of truth for MACRA display).
- `doc_presc` — primary e-prescribing table (new/active prescriptions, refill chains via `doc_presc_previous_id`).
- `current_medication` — separate medication reconciliation table, **not guaranteed to mirror `doc_presc`** (see Technical Findings).
- `patient_registration` — patient demographics (DOB, account number).
- `service_detail` + `cpt` — encounter billing/CPT data (used to confirm qualifying encounters for Initial Population/Denominator).
- `patient_assessments` — encounter-level diagnosis (assessment tab).
- `problem_list` — patient-level chronic diagnosis (assessment tab).
- `encounter` — encounter metadata (type, status, chargeable flag, comments).

### Application/Service Layer
- `MeasureCalcServiceImpl` — backend service responsible for measure calculation (previously implicated in unrelated bugs: `getMeasureRateReportByNPI`, `getCQMStatusByPatient` — see Related Prior Issues below).
- `getCQMStatusByPatient` — XHR endpoint (`glace-gwt.../getCQMStatusByPatient?patientID=...&providerId=...&accountId=malynew&year=2026...`) — supplies the JSON payload consumed by the MACRA Flowsheet UI.

### UI Layer (GWT Client)
- `MacraFlowsheetDesktopView.ui.xml` — pure view/layout definition (panels: `cmsMeasureStatusPanel`, `loadingCMSPanel`, etc.) — **contains no business logic**.
- `MacraFlowsheetDesktopView.java` — presenter/view class containing the **actual MET/NOT MET decision logic**, specifically the method `showCMSMeasureStatus(JSONObject response)`.

### Related Prior Issues (Context, Not This Ticket)
Per prior investigation history on this codebase (referenced for context, not re-litigated here):
- `MeasureCalcServiceImpl.getMeasureRateReportByNPI` — prior `ArrayIndexOutOfBoundsException` on `split("&&&")[1]` due to missing ECQM JSON cache files.
- `getCQMStatusByPatient` — prior 500 error caused by a 0-byte `236.json` cache file (freshness check trusted file age, not content validity).
- `EMeasureUtils.java` (`getInterventionQDM`, `getInterventionFromCNM`) — unrelated QRDA-I export bugs for CMS2v15 (measure 134).

These are **not confirmed to be related** to the current CMS156 issue, but they establish that the same measure-calculation and JSON-serialization pipeline (`measureInfo.get(measureId).toString().split("&&&")`) has had prior defects, and the `&&&`-delimited string parsing pattern recurs in the code discussed below.

---

## Investigation Timeline

1. **Initial complaint** — Doctor reports prescribing what he believes is a "high-risk medication" in 2026, but Criteria-1/Criteria-3 show NOT MET.
2. **Active medication panel reviewed** (chart UI screenshot) — medication list for patientId 8516 spans 2018–2026; 2026 entries: Warfarin Sodium (07/09/2026), Januvia, Omeprazole, Pantoprazole Sodium, Tamsulosin HCl (all 04/30/2026).
3. **First-pass valueset comparison** — none of the visibly "Active" 2026 meds matched any of the 17 CMS156 "Same High Risk Medications" drug classes (antihistamines, antiparkinsonians, GI antispasmodics, dipyridamole, guanfacine, nifedipine, specific antidepressants/barbiturates, ergoloid mesylates, meprobamate, estrogens, sulfonylureas, desiccated thyroid, nonbenzo hypnotics, skeletal muscle relaxants, specific pain meds, megestrol, meperidine) nor the Digoxin/Doxepin average-daily-dose branch, based on visible chart medications alone.
4. **Direct `doc_presc` query** (`WHERE doc_presc_patient_id = 8516 AND doc_presc_ordered_date BETWEEN '2026-01-01' AND '2026-12-31'`) revealed **three Digoxin orders not visible in the "Active Medications" chart summary**:
   - `doc_presc_id 107956` (2026-02-06) — `DIGOXIN 125 MCG TABLET`, **no RxNorm code populated**
   - `doc_presc_id 108006` (2026-02-09) — Digoxin, `rxnorm_cd = 197604`
   - `doc_presc_id 109854` (2026-04-30) — Digoxin, `rxnorm_cd = 197604`
5. **Dose calculation performed** against the CQL `MedicationStrengthPerUnit` function: RxNorm 197604 = "digoxin 0.125 MG Oral Tablet." All three orders: `90 tablets / 90 days = 0.125 mg/day` exactly.
6. **Compared against CQL threshold** (`"Average Daily Dose"(DigoxinOrdered) > 0.125 'mg/d'`) — **strict inequality**. 0.125 mg/d does **not** exceed 0.125 mg/d.
7. **Preliminary conclusion (later revisited):** Criteria-1/3 NOT MET appeared *clinically correct* — standard maintenance digoxin dosing does not cross the Beers-criteria supratherapeutic threshold that this measure branch targets.
8. **`current_medication` cross-check** — **zero digoxin rows** existed in `current_medication` for 2026, confirming this table and `doc_presc` diverge per-drug for this patient; any process reading `current_medication` instead of `doc_presc` would miss digoxin entirely.
9. **Follow-up `last_modified_date`/full-row dump of `doc_presc`** confirmed:
   - Orders 103315 (2025-08-17) → 108006 (2026-02-09) → 109854 (2026-04-30) form a **single refill chain** (`doc_presc_previous_id` linkage), each preserving the same drug/strength/days-supply.
   - Order 107956 (2026-02-06) is a **separate, out-of-chain order** created by a nurse (`doc_presc_med_internal_root_source = 'nurse'`), with **no RxNorm code populated** despite `doc_presc_is_uncoded = false`.
10. **`service_detail`/`cpt` join** confirmed three qualifying 99214 office visits in 2026 (01/15, 04/30, 07/21) → Initial Population/Denominator = 1 is correct and undisputed throughout.
11. **`patient_assessments`/`problem_list` check** — no relevant diagnoses found that would trigger exclusions in the reporting window relevant to this investigation.
12. **`quality_measures_patient_entries` direct query** (prompted by senior/Althaf's instruction to "check the code and mappings") revealed the **numerator values**:
    - Criteria 1: numerator = 1
    - Criteria 2: numerator = 0
    - Criteria 3: numerator = 1
    
    This **contradicted the dose-threshold analysis from step 6**, which predicted Criteria-1 numerator should be 0 (since no dose exceeded 0.125 mg/d). **This mismatch remains unresolved** (see Root Cause Analysis and Pending Work).
13. **UI vs DB comparison** — noted that UI-displayed statuses were the **exact logical inverse** of the raw DB numerators across all three criteria rows.
14. **Source code review** — `MacraFlowsheetDesktopView.ui.xml` inspected first; confirmed to be pure GWT UiBinder layout with no business logic.
15. **`MacraFlowsheetDesktopView.java` reviewed** — located the numerator-flip logic inside `showCMSMeasureStatus()`:
    ```java
    Boolean isInverseMeasure = Boolean.valueOf(measureInfo.get(measureId).toString()...split("&&&")[1]);
    ...
    if (isInverseMeasure) {
        numerator = (numerator == 0 ? 1 : 0);
    }
    ```
16. **Final reconciliation** — re-applying the flip to all three DB rows reproduced the UI exactly:

    | Criteria | DB Numerator | Post-flip | Denominator | UI Status |
    |---|---|---|---|---|
    | 1 | 1 | 0 | 1 | NOT MET ✓ |
    | 2 | 0 | 1 | 1 | MET ✓ |
    | 3 | 1 | 0 | 1 | NOT MET ✓ |

    This **fully explained** the UI/DB "mismatch" as intentional design, and also explained why Criteria-2 MET + Criteria-3 NOT MET only *looked* impossible — the raw DB values (`Numerator2=0, Numerator3=1`) are **internally consistent** with the CQL spec (`Numerator3 = Numerator1 OR Numerator2 = 1 OR 0 = 1`, i.e., MET before the flip is even applied).
17. Findings communicated to senior (Althaf) via WhatsApp summary, including the still-open dosage/numerator discrepancy as the next investigation thread.

---

## Root Cause Analysis

### What was initially suspected (and ruled out)
- **Suspected:** A bug in the MACRA UI where MET/NOT MET labels were rendered backwards relative to the calculated numerator, or a criteria-index/mapping error between the DB and the UI.
- **Ruled out:** Code review confirmed the UI's numerator-flip is a **deliberate, consistently-applied transformation** for measures flagged `isInverseMeasure = true`, not a bug. All three criteria rows reconcile perfectly once the flip is accounted for.

### Confirmed root cause of the "UI shows opposite of DB" appearance
CMS156v13 is defined by NCQA/CMS as an **inverse measure** (`Improvement Notation: Lower score indicates better quality`). The raw CQL numerator represents "patient received the risky medication combination" — a **negative clinical outcome**. To keep the visual convention consistent across all measures (green "MET" = good, red "NOT MET" = bad, regardless of measure direction), the GWT client (`MacraFlowsheetDesktopView.java`) inverts the numerator for any measure carrying the `isInverseMeasure` flag **before** computing the MET/NOT MET/PARTIALLY MET/EXCLUSION label. This is **not a defect** in itself.

> ⚠️ **Design risk noted (not confirmed as a bug, but flagged for review):** The inversion is applied purely at the **client rendering layer**, conflating two logically distinct concepts:
> 1. "This measure scores in reverse for MIPS percentage/points purposes" (a **scoring/reporting** concern)
> 2. "Flip the boolean label shown per-criteria" (a **display** concern)
>
> If any other consumer of `quality_measures_patient_entries` (e.g., QRDA-I export, MIPS submission file generation, other dashboards) does **not** apply the same inversion, there is a risk of **inconsistent MET/NOT MET semantics across the product** for inverse measures. This was not verified in this investigation and should be checked (see Pending Work).

### Unresolved / suspected secondary defect (requires further investigation)
Independent of the display-inversion finding, a **separate, still-open discrepancy** was identified in the **calculation engine's numerator value itself**:

- All three of patient 8516's 2026 digoxin orders resolve to **exactly 0.125 mg/day** (90 tablets ÷ 90 days × 0.125 mg/tablet), per RxNorm code 197604 mapped via the CQL's `MedicationStrengthPerUnit` function.
- The CQL specification requires **strictly greater than** 0.125 mg/d (`> 0.125 'mg/d'`) for the Digoxin branch of "High Risk Medications with Average Daily Dose Criteria" to trigger.
- Based on this, Criteria-1's raw numerator (pre-flip) was expected to be **0** (no qualifying event) — no Digoxin order exceeds the threshold, and Digoxin does not appear in the "Same High Risk Medications Ordered on Different Days" branch (17 unions) or the antiinfective-based "Prolonged Duration" branch.
- However, `quality_measures_patient_entries` shows Criteria-1 raw numerator = **1**.

**Possible explanations (not yet confirmed):**
1. A `>=` vs `>` comparison bug in the backend's dose-threshold implementation (i.e., the calc engine may be using `>=` instead of the spec's `>`).
2. Some other high-risk medication/branch not yet checked is legitimately triggering Numerator 1 (i.e., a drug outside Digoxin that was missed in this investigation's scope, since the investigation focused specifically on Digoxin).
3. Stale/cached calculation results in `quality_measures_patient_entries` that predate a data correction (refill chain edits, code corrections, etc.) and have not been recalculated.
4. The uncoded 2026-02-06 order (`doc_presc_id 107956`, no RxNorm code) being handled by a different code path than expected (e.g., matched by drug-name string rather than code, which could behave inconsistently with the coded orders).

**This is the primary unresolved technical question and the next concrete investigation step** (see Pending Work).

### Contributing/observed data-quality issue (separate from the MET/NOT MET question)
- `doc_presc_id 107956` (2026-02-06, "DIGOXIN 125 MCG TABLET") has **both `doc_presc_rxnorm_code` and `doc_presc_rxnorm_cd` blank**, despite `doc_presc_is_uncoded = false`. This order would be **invisible to any RxNorm-code-based matching logic** (which the CQL spec requires), regardless of what triggered Criteria-1's numerator. This is a **data-entry/integration gap**, most likely originating from the nurse-entry workflow (`doc_presc_med_internal_root_source = 'nurse'`), and is a risk for **any** eCQM that keys off RxNorm codes, not just CMS156.

---

## Detailed Technical Findings

### Medication data for Patient 8516, Reporting Year 2026 (`doc_presc`)

| doc_presc_id | Drug | RxNorm (`rxnorm_cd`) | Ordered Date | Qty/Days | Avg Daily Dose | Notes |
|---|---|---|---|---|---|---|
| 107956 | DIGOXIN 125 MCG TABLET | *(blank)* | 2026-02-06 | 90/90 | 0.125 mg/d | Uncoded; separate order from nurse; `doc_presc_previous_id = 103315` |
| 108006 | Digoxin | 197604 | 2026-02-09 | 90/90 | 0.125 mg/d | Refill of 103315 (2025-08-17); `previous_id = 103315` |
| 109854 | Digoxin | 197604 | 2026-04-30 | 90/90 | 0.125 mg/d | Refill of 108006; `previous_id = 108006` |
| — | Warfarin Sodium 2.5mg | — | 2026-07-09 | 30 days | — | Not on any CMS156 valueset |
| — | Januvia (sitagliptin) 50mg | 665042 | 2026-04-30 | 30 days | — | Not on any CMS156 valueset |
| — | Omeprazole 40mg | 200329 | 2026-04-30 | 90 days | — | Not on any CMS156 valueset |
| — | Pantoprazole Sodium 20mg | 251872 | 2026-04-30 | — | — | Not on any CMS156 valueset |
| — | Tamsulosin HCl 0.4mg | 863669 | 2026-04-30 | — | — | Not on any CMS156 valueset |

**Note:** Levothyroxine (2025), Montelukast (2025), Gabapentin (2025), Losartan (2024), Metoprolol (2022), Magnesium Oxide (2022) all fall **outside** the 2026 measurement period and were excluded from analysis on that basis.

### Qualifying Encounters (Denominator/Initial Population confirmation)

```
service_detail_dos | cpt_cptcode | cpt_description
2026-01-15          | 99214       | Office/outpatient visit
2026-04-30          | 99214       | Office/outpatient visit
2026-07-21          | 99214       | Office/outpatient visit
```
Three qualifying 99214 visits confirm Denominator = 1 (Initial Population met via age ≥65 + qualifying encounter). **Not in dispute at any point in this investigation.**

### `current_medication` vs `doc_presc` divergence
`current_medication` for patient 8516, 2026, contains **no digoxin entries at all** — confirming these two tables are populated independently per drug and are **not guaranteed to be synchronized**. Any reporting/calc process must be verified to read from the correct source table for each measure's data needs.

### `quality_measures_patient_entries` raw values (ground truth prior to UI transformation)

```
 patient_id | reporting_year | measure_id | criteria | ipp | denominator | denominator_exclusion | numerator | numerator_exclusion | denominator_exception
       8516 |            2026 |        238 |        1 |   1 |           1 |                      0 |         1 |                    0 |                      0
       8516 |            2026 |        238 |        2 |   1 |           1 |                      0 |         0 |                    0 |                      0
       8516 |            2026 |        238 |        3 |   1 |           1 |                      0 |         1 |                    0 |                      0
```

### UI Rendering Logic — `MacraFlowsheetDesktopView.java`, `showCMSMeasureStatus()`

Key extracted logic (annotated):

```java
// Parse per-measure metadata; measureInfo values are "&&&"-delimited strings:
// [0] = measure name, [1] = isInverseMeasure flag
String measureName = measureInfo.get(measureId).toString()...split("&&&")[0];
Boolean isInverseMeasure = Boolean.valueOf(
    measureInfo.get(measureId).toString()...split("&&&")[1]
);

int denominator = Integer.parseInt(eachMeasureObject.get("denominator").toString());
int exclusion   = Integer.parseInt(eachMeasureObject.get("denominatorExclusion").toString());
int numerator   = Integer.parseInt(eachMeasureObject.get("numerator").toString());
int exception   = Integer.parseInt(eachMeasureObject.get("denominatorException").toString());

// *** THE INVERSION LOGIC ***
if (isInverseMeasure) {
    numerator = (numerator == 0 ? 1 : 0);
}

// Status decision, AFTER inversion is applied:
if (denominator == 0)                              status = "N/A";
else if (exclusion > 0)                            status = "EXCLUSION";
else if (denominator == numerator)                 status = "MET";
else if (numerator > 0 && denominator > numerator) status = "PARTIALLY MET";
else if (exception > 0)                             status = "EXCEPTION";
else                                                status = "NOT MET";
```

**Key observation:** The inversion is applied **once**, uniformly, to every criteria row of any measure carrying `isInverseMeasure = true`. There is no per-criteria special-casing — meaning the mechanism itself is structurally sound and not the source of the reported "impossible" Criteria-2/3 combination (that combination is fully explained once the flip is understood).

**Unverified item:** This investigation did not directly inspect the `measureInfo` payload's `&&&`-delimited string for measure 238 to **positively confirm** the `isInverseMeasure` flag is `true` in this dataset — this was inferred from CMS156's documented "Improvement Notation: Lower score indicates better quality" and from the fact that applying the flip perfectly reproduces the observed UI output. Direct confirmation via the network payload (see Investigation Timeline, `getCQMStatusByPatient` XHR call) is recommended to close this out definitively.

---

## SQL Analysis and Scripts

All SQL below is **read-only, investigation-only**. No fix scripts, migrations, or data-modification queries were executed or proposed during this investigation.

### Investigation Queries

**1. Confirm patient identity/DOB**
```sql
SELECT
    patient_registration_id,
    patient_registration_accountno,
    patient_registration_dob
FROM patient_registration
WHERE patient_registration_accountno = '008516';
```
- **Purpose:** Confirm `patientId 8516` maps to account `008516` and validate age-eligibility (DOB 1937-08-26 → ≥65 in measurement year 2026).
- **Tables:** `patient_registration`
- **Impact:** Read-only, no risk.

**2. Digoxin/medication orders in 2026 (`doc_presc`)**
```sql
SELECT
    doc_presc_id,
    doc_presc_rx_name,
    doc_presc_rxnorm_code,
    doc_presc_rxnorm_cd,
    doc_presc_ordered_date,
    doc_presc_start_date,
    doc_presc_status,
    doc_presc_is_active
FROM doc_presc
WHERE doc_presc_patient_id = 8516
  AND doc_presc_ordered_date BETWEEN '2026-01-01' AND '2026-12-31'
ORDER BY doc_presc_ordered_date;
```
- **Purpose:** Enumerate all e-prescribing orders for the patient within the measurement period.
- **Tables:** `doc_presc`
- **Indexing:** Uses `doc_presc_patient_id_index` and benefits from `doc_presc_ordered_date_idx`; acceptable performance for single-patient lookups.
- **Notes:** `doc_presc_ordered_date` and `doc_presc_last_modified_date` can diverge significantly (see refill-chain analysis below) — always confirm which timestamp column the calc engine actually treats as `authorDatetime` per the CQL spec.

**3. Same query filtered by `last_modified_date` (used to trace refill-chain history)**
```sql
SELECT
    doc_presc_id,
    doc_presc_rx_name,
    doc_presc_rxnorm_code,
    doc_presc_rxnorm_cd,
    doc_presc_ordered_date,
    doc_presc_start_date,
    doc_presc_last_modified_date,
    doc_presc_status,
    doc_presc_is_active
FROM doc_presc
WHERE doc_presc_patient_id = 8516
  AND doc_presc_last_modified_date BETWEEN '2026-01-01' AND '2026-12-31'
ORDER BY doc_presc_last_modified_date;
```
- **Purpose:** Reveal the full refill chain, including the 2025-08-17 original order (103315) that was modified/superseded within 2026, which the `ordered_date`-only query would have missed.
- **Finding:** Confirmed `doc_presc_previous_id` chain: 103315 → 108006 → 109854.

**4. Medication reconciliation table cross-check**
```sql
SELECT
    current_medication_id,
    current_medication_rx_name,
    current_medication_rxnorm_code,
    current_medictiion_rxnorm_code,   -- NOTE: misspelled column name, exists as-is in schema
    current_medication_rxnorm_cd,
    current_medication_order_on,
    current_medication_modified_on,
    current_medication_start_date,
    current_medication_status,
    current_medication_is_active
FROM current_medication
WHERE current_medication_patient_id = 8516
  AND current_medication_modified_on BETWEEN '2026-01-01' AND '2026-12-31'
ORDER BY current_medication_modified_on;
```
- **Purpose:** Cross-check whether `current_medication` independently reflects the same drugs as `doc_presc`.
- **Finding:** **No digoxin rows present** — confirms table divergence for this drug/patient.
- ⚠️ **Schema note:** The column `current_medictiion_rxnorm_code` contains a typo (extra "ti") baked into the live schema — always reference exact column name, do not "correct" it in queries.

**5. Full-column dump of specific `doc_presc` rows (root-cause deep dive)**
```sql
SELECT *
FROM doc_presc
WHERE doc_presc_patient_id = 8516
  AND doc_presc_last_modified_date BETWEEN '2026-01-01' AND '2026-12-31'
ORDER BY doc_presc_last_modified_date;
```
- **Purpose:** Full-row inspection to check every metadata field (encounter linkage, `previous_id`, `rxnorm_cd`, pharmacy info, e-prescribing status) for the digoxin refill chain.
- **Key fields validated:** `doc_presc_encounter_id`, `doc_presc_previous_id`, `doc_presc_previous_source`, `doc_presc_is_uncoded`, `doc_presc_rxnorm_cd`.

**6. Qualifying encounters / Denominator confirmation**
```sql
SELECT sd.service_detail_dos, c.cpt_cptcode, c.cpt_description
FROM service_detail sd
JOIN cpt c ON c.cpt_id = sd.service_detail_cptid
WHERE sd.service_detail_patientid = 8516
  AND sd.service_detail_dos BETWEEN '2026-01-01' AND '2026-12-31'
ORDER BY sd.service_detail_dos;
```
- **Purpose:** Verify Initial Population/Denominator via qualifying CPT-coded encounters.
- **Join:** `service_detail.service_detail_cptid = cpt.cpt_id` (standard FK-style join, no fan-out risk given CPT is a lookup table).
- **Finding:** Three 99214 visits confirmed (01/15, 04/30, 07/21/2026) plus "OVERRIDE CPT CODE" (002) entries alongside each — denominator logic not in question.

**7. Assessment/Diagnosis checks (exclusion validation)**
```sql
SELECT * FROM patient_assessments
WHERE patient_assessments_patientid = 8516
  AND patient_assessments_encounterdate BETWEEN '2026-01-01' AND '2026-12-31'
ORDER BY patient_assessments_encounterdate;

SELECT * FROM problem_list
WHERE problem_list_patient_id = 8516
  AND problem_list_last_mod_on BETWEEN '2026-01-01' AND '2026-12-31'
ORDER BY problem_list_last_mod_on;
```
- **Purpose:** Check for Hospice/Palliative Care diagnoses (denominator exclusions) and any diagnosis relevant to Numerator 2 (antipsychotic/benzodiazepine treated-diagnosis exceptions).
- **Finding:** `patient_assessments` returned one unrelated BMI diagnosis (Z68.26); `problem_list` returned zero rows for the period. No exclusion-triggering diagnoses found.

**8. Encounter metadata for prescribing events**
```sql
SELECT *
FROM encounter
WHERE encounter_id IN (
    SELECT doc_presc_encounter_id
    FROM doc_presc
    WHERE doc_presc_patient_id = 8516
      AND DATE(doc_presc_last_modified_date) BETWEEN '2026-01-01' AND '2026-12-31'
)
ORDER BY encounter_date;
```
- **Purpose:** Determine whether the prescribing encounters are billable/qualifying visits or purely administrative refill records.
- **Finding:** `encounter_id 446985` (02/09/2026 refill) is a **non-chargeable, refill-only pseudo-encounter** (`encounter_chargeable = false`, `encounter_comments = 'Prescription'`). This does **not** affect the denominator, since the denominator was already independently satisfied by the three 99214 visits — but it is a useful pattern to recognize when auditing "qualifying encounter" logic in general (refill-triggered encounters should not be miscounted as qualifying office visits elsewhere in the system).

**9. Ground-truth calculation table**
```sql
SELECT
    quality_measures_patient_entries_patient_id      AS patient_id,
    quality_measures_patient_entries_reporting_year  AS reporting_year,
    quality_measures_patient_entries_measure_id      AS measure_id,
    quality_measures_patient_entries_criteria        AS criteria,
    quality_measures_patient_entries_ipp             AS ipp,
    quality_measures_patient_entries_denominator     AS denominator,
    quality_measures_patient_entries_denominator_exclusion AS denominator_exclusion,
    quality_measures_patient_entries_numerator       AS numerator,
    quality_measures_patient_entries_numerator_exclusion AS numerator_exclusion,
    quality_measures_patient_entries_denominator_exception AS denominator_exception,
    quality_measures_patient_entries_provider_id     AS provider_id,
    quality_measures_patient_entries_npi             AS npi,
    quality_measures_patient_entries_tin             AS tin,
    quality_measures_patient_entries_updated_on      AS updated_on
FROM quality_measures_patient_entries
WHERE quality_measures_patient_entries_patient_id = 8516
  AND quality_measures_patient_entries_measure_id = '238'
  AND quality_measures_patient_entries_reporting_year = 2026
ORDER BY quality_measures_patient_entries_criteria;
```
- **Purpose:** Retrieve the **authoritative, pre-display-transformation** calculation output for direct comparison against the UI.
- **This is the single most important query in the investigation** — it is what surfaced the raw numerator values that, once reconciled against the UI inversion logic, resolved the apparent discrepancy.
- **Recommended reuse:** This query template should be the **first step** for any future "MACRA UI looks wrong" ticket — always compare raw DB numerator against UI status before assuming either is wrong.

### Fix / Migration / Cleanup Scripts
**None.** No data corrections, migrations, or cleanup scripts were executed or proposed in this investigation. The one identified data-quality issue (uncoded order `doc_presc_id 107956`) has **not** been corrected and remains a manual data-entry gap requiring separate remediation (see Pending Work).

---

## Code Changes

**No code changes were made during this investigation.** This was a read-only diagnostic/RCA exercise. The following files were **reviewed only**:

| File | Path | Role | Outcome |
|---|---|---|---|
| `MacraFlowsheetDesktopView.ui.xml` | `.../client/application/chart/macra/` | GWT UiBinder layout | Confirmed no business logic present; purely structural. |
| `MacraFlowsheetDesktopView.java` | `.../client/application/chart/macra/` | GWT Presenter/View — `showCMSMeasureStatus()` | **Root cause located**: numerator inversion logic for `isInverseMeasure` measures, confirmed working as designed. |

No modification to either file is recommended based on findings to date. If the backend numerator discrepancy (see Root Cause Analysis) is confirmed as a genuine calc bug, the fix would belong in the **backend calculation service** (likely `MeasureCalcServiceImpl` or a related class responsible for populating `quality_measures_patient_entries`), **not** in the GWT client reviewed here.

---

## Fixes and Workarounds

| Type | Status |
|---|---|
| Workaround | None applied — issue was explained as intended UI behavior for the display-inversion component. |
| Permanent Fix | Not yet required for the display layer (working as designed). **Pending** for the suspected backend numerator/dose-threshold discrepancy. |
| Config Change | None. |
| Data Correction | **Not yet performed** — the uncoded `doc_presc_id 107956` order should be corrected (RxNorm code backfilled) by whoever owns nurse Rx-entry data quality, but this was not actioned in this investigation. |
| Reprocessing | If a backend calc bug is confirmed, `quality_measures_patient_entries` for measure 238 may need to be **recalculated/reprocessed** for affected patients/tenants once the fix is deployed. Scope of "affected patients" is currently unknown (see Pending Work). |

---

## Validation and Testing

- **Manual SQL cross-validation** performed between `doc_presc`, `current_medication`, `quality_measures_patient_entries`, and the CQL specification's dosage functions — this constitutes the primary validation method used.
- **UI-vs-DB reconciliation table** (see Investigation Timeline, step 16) validated the inversion-logic hypothesis with 100% match across all three criteria rows for this patient.
- **Not yet performed / recommended for closure:**
  - Direct inspection of the `getCQMStatusByPatient` JSON response body (via browser DevTools → Network → Response) to **positively confirm** `isInverseMeasure = true` for measure 238 in the live payload, rather than relying on inference.
  - Re-running the full "Same High Risk Medications" 17-valueset check (not just Digoxin) against this patient's complete 2026 medication list, to rule out an alternate drug/branch as the source of Criteria-1's numerator=1.
  - Testing whether other patients/tenants with confirmed digoxin dosing **above** 0.125 mg/d correctly produce numerator=1 pre-flip (positive control) and whether patients at exactly 0.125 mg/d consistently produce numerator=0 pre-flip (negative control) — this would definitively confirm or refute the suspected `>=` vs `>` comparison bug.

---

## Risks and Side Effects

- **Data integrity risk (unconfirmed):** If the suspected `>=`/`>` threshold bug is real, it could be **systemically affecting all patients** on this measure (and potentially other measures with similar strict-inequality dose thresholds, e.g., Doxepin's `> 6 'mg/d'` branch), not just patient 8516. This has MIPS scoring accuracy implications across the tenant base if confirmed.
- **Cross-consumer consistency risk (unconfirmed):** The `isInverseMeasure` flip is applied only in this one GWT view class. Any other system that reads `quality_measures_patient_entries` directly (QRDA-I export, MIPS submission files, other dashboards/reports) must independently apply the same inversion logic for inverse measures, or risk showing **inconsistent MET/NOT MET semantics** across the product for the same underlying data. This was **not verified** during this investigation.
- **Data quality risk (confirmed, low severity so far):** Uncoded prescription orders (missing RxNorm) from nurse-entry workflows are invisible to any code-based eCQM matching logic — this is a general risk across **all** RxNorm-dependent measures, not unique to CMS156.
- **No production risk from this investigation itself** — all activity was read-only.

---

## Pending Work

1. **[HIGH PRIORITY]** Confirm whether the backend calc engine correctly implements `> 0.125 'mg/d'` (strict) for the Digoxin average-daily-dose branch, or whether it is incorrectly using `>=`. Requires code review of the measure-238 calculation implementation (likely in `MeasureCalcServiceImpl` or equivalent) and/or targeted test-patient validation.
2. **[HIGH PRIORITY]** Determine the actual source of Criteria-1's raw numerator = 1 for patient 8516 — run the full 17-valueset "Same High Risk Medications" check plus the antiinfective "Prolonged Duration" branch against the patient's complete 2026 medication list (not just Digoxin), to rule out an alternate trigger.
3. **[MEDIUM]** Directly inspect the `getCQMStatusByPatient` JSON response for measure 238 to positively confirm the `isInverseMeasure` flag value, closing out the one inferred-but-unverified assumption in this RCA.
4. **[MEDIUM]** Audit whether other consumers of `quality_measures_patient_entries` (QRDA-I export, MIPS submission generation, other reporting dashboards) correctly apply the same inverse-measure flip, to prevent cross-system MET/NOT MET inconsistency.
5. **[LOW]** Backfill the missing RxNorm code on `doc_presc_id 107956` (or establish a validation rule preventing uncoded Rx entries from nurse workflows going forward).
6. **[LOW]** If the backend threshold bug is confirmed and fixed, assess scope of impact and determine whether historical `quality_measures_patient_entries` rows need bulk recalculation for the current reporting year across affected tenants.

---

## Lessons Learned

### Debugging Insights
- **Always separate "display logic" from "calculation logic" before concluding a bug exists.** What initially looked like a severe, logically-impossible UI/DB mismatch was fully explained by a single, consistently-applied client-side transformation. Jumping to "the code is broken" without first locating the transformation layer would have wasted significant effort investigating a non-issue.
- **For inverse/negative CMS measures, always check the measure's "Improvement Notation" before interpreting MET/NOT MET.** A raw numerator of 1 does not always mean "good" — for measures like CMS156, it usually means the opposite.
- **When a UI value doesn't match a DB value, get the raw DB value first, then find the exact code path between DB and screen** — this query (`quality_measures_patient_entries` direct select) should be the default first step for any "the MACRA tab looks wrong" ticket.
- **Cross-check both `doc_presc` and `current_medication`** whenever a medication seems "missing" from the calculation — these tables are not guaranteed to be synchronized per drug, and support/dev staff should not assume one mirrors the other.
- **Query by both `ordered_date` and `last_modified_date`** when investigating refill chains — a query scoped only to `ordered_date` can miss orders that were modified/refilled within the period even though originally created earlier.

### Prevention Ideas
- Add a **unit/regression test** at the calc-engine level asserting strict-inequality dose thresholds (`> X`, not `>= X`) for every measure branch that specifies "exceeding" a threshold, to prevent silent `>=`/`>` substitution bugs.
- Add a **data-entry validation rule** requiring an RxNorm code before a prescription order can be saved (or at minimum, flag/queue uncoded orders for pharmacist/coder review), especially from nurse-entry workflows.
- Consider **centralizing the `isInverseMeasure` inversion logic** in a shared utility/service rather than duplicating it per UI class, so any additional consumer of quality-measure data (exports, dashboards, reports) automatically inherits correct inverse-measure handling and cannot diverge from the canonical MET/NOT MET semantics.

### Monitoring Recommendations
- Add logging/alerting around the calc engine when a numerator is computed for a dose-threshold branch, capturing the actual computed average-daily-dose value alongside the pass/fail decision — this would have made this investigation immediate rather than requiring manual dose reconstruction from raw prescription rows.
- Consider a periodic data-quality report flagging prescriptions with missing RxNorm codes, segmented by entry source (`doc_presc_med_internal_root_source`), to proactively surface nurse-workflow coding gaps like the one found on `doc_presc_id 107956`.

### Process Improvements
- When escalating a "quality measure shows wrong status" ticket, the standard triage checklist should include, in order:
  1. Pull raw `quality_measures_patient_entries` values.
  2. Check the measure's CMS "Improvement Notation" (inverse vs. standard).
  3. Compare raw numerator against UI status, applying the inversion if applicable, **before** assuming either the calc engine or the UI is at fault.
  4. Only after reconciling display logic, drill into whether the *raw* numerator itself is clinically/technically correct per the CQL spec.
