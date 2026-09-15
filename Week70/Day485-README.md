# MIPS CMS68v6 Medication Review – Provider/Encounter Mismatch Investigation

## Overview

This document captures the investigation of a reported **MIPS medication review issue** in the AGC/Allergy Consultant environment.

The reported behavior was:

* A user reviewed current medications in the patient chart.
* The UI showed the medications as **Reviewed**.
* The overall MIPS performance percentage increased after the review.
* However, when the user opened the MIPS **Not Met** patient list, the same patient continued to appear as **Not Met / Not Reviewed**.
* The investigation identified that the medication review was recorded against an encounter belonging to **Charlene Shookoff M.D. (Provider ID 3308)**, while the MIPS report was being evaluated for **Robert Schramm M.D. (Provider ID 17)**.
* Robert Schramm is configured for **2026 MIPS Measure 130 / CMS68v6**.
* Charlene Shookoff is **not configured for 2026 MIPS** and does not have Measure 130 configured.

The investigation has established the relevant database state and provider configuration. However, the **code-level/business-rule root cause is not fully confirmed** until the CMS68 calculation and patient-detail logic are traced.

---

## Background / Context

### Reported customer issue

The support email reported:

> The user reviewed a number of current medications on Thursday and saw an increase in the MIPS completion percentage. Later, the same patients still appeared in the Not Met list even though their medications were marked as Reviewed in the patient chart.

### Report

**Subject:** `MIPS Issue/agc`

**Reported:** September 15, 2026

**Reported functionality:** MIPS → Quality Measures → Current Medication Review

### Affected MIPS measure

The investigation identified the relevant measure as:

| Field               | Value                                                          |
| ------------------- | -------------------------------------------------------------- |
| Internal Measure ID | **130**                                                        |
| CMS ID              | **CMS68v6**                                                    |
| Measure             | **Documentation of Current Medications in the Medical Record** |
| Reporting Year      | **2026**                                                       |
| Submission Type     | EHR                                                            |

---

# Problem Statement

The MIPS report shows an affected patient as **Not Met / Not Reviewed**, even though the patient's Current Medications screen shows the medications as **Reviewed**.

For the investigated patient:

```text
Patient ID : 8109
Chart ID   : 8346
Account    : 008109
Patient    : GREYSON ADAMS
```

The relevant encounter date is:

```text
08/19/2026
```

There are **two encounters on the same date**, belonging to different providers.

```text
Encounter 161653
Provider: 3308 - Charlene Shookoff M.D.
Medication attestation: 428191000124101
                         → Reviewed

Encounter 161660
Provider: 17 - Robert Schramm M.D.
Medication attestation: NULL
                         → Not Reviewed
```

The MIPS report is for:

```text
Provider: Robert Schramm M.D.
Provider ID: 17
Measure: 130 / CMS68v6
```

Therefore, the medication review exists on an encounter belonging to a **different provider** from the provider whose MIPS measure is being evaluated.

---

# Impact

The observed impact is:

* Patients can remain in the MIPS **Not Met** list.
* The patient-level MIPS status may show **Not Reviewed**.
* The chart UI can simultaneously show **Reviewed**.
* The overall MIPS performance percentage can increase after medication reviews, according to the customer report.
* This creates an apparent inconsistency between:

  * the clinical chart,
  * the encounter data,
  * and the MIPS patient-level report.

### Reported performance

The screenshot showed:

```text
IPP          : 1469
Denominator  : 1469
Numerator    : 109
Performance  : 7.42%
```

The customer also reported that the percentage increased after performing medication reviews.

The screenshot annotation indicated a progression approximately from:

```text
1.80% → 3% → 7.42%
```

This progression was observed from the provided screenshots/notes and was not independently recalculated during the investigation.

---

# Environment Details

## Application

The investigation concerns the Glace EMR/MIPS functionality.

Relevant UI areas:

* EMR
* Patient Chart
* Current Medications
* Quality Measures
* MIPS report
* Patient-level MIPS status popup

The provided patient URL was:

```text
Patient ID: 8109
Chart ID: 8346
```

The URL shown in the screenshot was a Patient Billing Tracker URL containing those identifiers. The billing tracker itself was used to inspect the patient's provider/service information; no billing defect was established.

---

# System Components Involved

## 1. Current Medications UI

The patient chart contains a **Current Medications** section with a control showing:

```text
Yes
Reviewed
```

The screenshot showed the medication review as completed.

---

## 2. Encounter

The `encounter` table stores the encounter-level medication attestation.

Important columns:

```text
encounter_id
encounter_chartid
encounter_date
encounter_service_doctor
encounter_med_review
medication_attestation_status
encounter_created_by
encounter_created_date
encounter_modifiedby
encounter_modifiedon
```

The investigation established that `medication_attestation_status` is the relevant field for determining whether the Reviewed SNOMED code is present.

Per the investigation/business rule supplied:

```text
medication_attestation_status contains Reviewed SNOMED
    → Reviewed

medication_attestation_status is NULL/empty
    → Not Reviewed
```

The exact semantic definition of the SNOMED code should still be confirmed against the application/value-set configuration.

---

## 3. MIPS Provider Configuration

Table:

```text
macra_provider_configuration
```

This determines whether a provider has MIPS configuration for a reporting year.

Important columns:

```text
macra_provider_configuration_provider_id
macra_provider_configuration_reporting_year
macra_provider_configuration_reporting_start
macra_provider_configuration_reporting_end
macra_provider_configuration_reporting_method
macra_provider_configuration_report_type
```

---

## 4. Provider Measure Mapping

Table:

```text
quality_measures_provider_mapping
```

This determines which quality measures are configured for a provider for a reporting year.

Important columns:

```text
quality_measures_provider_mapping_provider_id
quality_measures_provider_mapping_reporting_year
quality_measures_provider_mapping_measure_id
quality_measures_provider_mapping_cmsid
quality_measures_provider_mapping_title
quality_measures_provider_mapping_priority
```

---

## 5. Measure Definition

Table:

```text
measure_details
```

This contains the global measure definition:

```text
measure_id
cms_id
title
```

For Measure `130`:

```text
Measure ID: 130
CMS ID:     CMS68v6
Title:      Documentation of Current Medications in the Medical Record
```

---

## 6. Employee/Provider Profile

Table:

```text
emp_profile
```

Relevant provider identity fields include:

```text
emp_profile_empid
emp_profile_doctorid
emp_profile_fullname
emp_profile_loginid
emp_profile_provider_map_id
```

### Important finding

An attempted join using:

```sql
CAST(emp_profile_doctorid AS INTEGER)
```

failed because `emp_profile_doctorid` contains non-numeric values such as:

```text
G235
```

Therefore, `emp_profile_doctorid` must **not** be blindly cast to integer.

The provider IDs used by the MIPS configuration were instead validated directly using the provider IDs from the application/provider dropdown and the MIPS configuration tables.

---

# Investigation Timeline

## Step 1 – Review customer report

The customer reported that:

1. Current medications were reviewed.
2. MIPS percentage increased.
3. The patient later remained in the Not Met list.
4. The MIPS patient detail showed Not Reviewed.
5. The chart showed Reviewed.

This established a discrepancy between the chart and MIPS reporting.

---

## Step 2 – Identify medication status storage

The `current_medication` table was inspected.

The table contains:

```text
current_medication_status
current_medication_is_active
current_medication_encounter_id
current_medication_patient_id
...
```

A status distribution query returned:

```text
status | count
-------+------
11     | 1
14     | 46324
15     | 53028
16     | 864
18     | 25
20     | 2414
```

Initially, `current_medication_status` was considered as a possible indicator of medication review.

### Finding

`current_medication_status` alone could not be established as the source of the MIPS Reviewed/Not Reviewed status.

The investigation subsequently identified:

```text
encounter.medication_attestation_status
```

as the relevant encounter-level field.

---

## Step 3 – Initial medication investigation

An initial patient ID of `17621` was used while examining the `current_medication` table.

That patient had multiple medication records with statuses `14`, `15`, and `16`.

This was useful for understanding the medication table but was **not the final affected patient from the support screenshots**.

The actual affected patient was later identified as:

```text
patientId = 8109
chartId   = 8346
```

---

## Step 4 – Identify MIPS measure

Provider configuration and measure mapping were inspected.

The relevant measure was identified as:

```text
Measure ID: 130
CMS ID: CMS68v6
Documentation of Current Medications in the Medical Record
```

---

## Step 5 – Verify MIPS configuration

The provider dropdown contained:

```text
3308 → Charlene Shookoff M.D
1442 → Glenwood undefined
17   → Robert Schramm M.D.
1    → Test Doctor M.D.
```

The 2026 MIPS configuration was then checked.

For the relevant providers:

```text
Provider 17
    → MIPS configured
    → Measure 130 configured

Provider 3308
    → MIPS not configured
    → Measure 130 not configured
```

---

## Step 6 – Verify Robert Schramm's 2026 measures

Robert Schramm (`17`) has the following 2026 mappings:

| Measure ID | CMS ID      | Measure                                                                                   |
| ---------: | ----------- | ----------------------------------------------------------------------------------------- |
|         65 | CMS154v5    | Appropriate Treatment for Children with Upper Respiratory Infection (URI)                 |
|        128 | CMS69v5     | Preventive Care and Screening: Body Mass Index (BMI) Screening and Follow-Up Plan         |
|    **130** | **CMS68v6** | **Documentation of Current Medications in the Medical Record**                            |
|        226 | CMS138v5    | Preventive Care and Screening: Tobacco Use: Screening and Cessation Intervention          |
|        236 | CMS165v5    | Controlling High Blood Pressure                                                           |
|        238 | CMS156v5    | Use of High-Risk Medications in the Elderly                                               |
|        317 | CMS22v5     | Preventive Care and Screening: Screening for High Blood Pressure and Follow-Up Documented |
|        374 | CMS50v5     | Closing the Referral Loop: Receipt of Specialist Report                                   |
|   IA_AHE_6 | —           | Provide Education Opportunities for New Clinicians                                        |
|   IA_EPA_2 | —           | Use of telehealth services that expand practice access                                    |
|   IA_PM_13 | —           | Chronic Care and Preventative Care Management for Empaneled Patients                      |

Thus:

```text
Robert Schramm
Provider ID: 17
MIPS 2026: YES
CMS68 / Measure 130: YES
```

---

## Step 7 – Identify affected encounters

For patient:

```text
Patient ID = 8109
Chart ID   = 8346
Date       = 08/19/2026
```

two encounters were found:

### Encounter 161653

```text
Provider:
3308 - Charlene Shookoff M.D.

medication_attestation_status:
428191000124101
```

Per the investigation rule, this represents:

```text
REVIEWED
```

### Encounter 161660

```text
Provider:
17 - Robert Schramm M.D.

medication_attestation_status:
NULL
```

This represents:

```text
NOT REVIEWED
```

---

# Root Cause Analysis

## Confirmed data-level root cause / condition

The strongest confirmed finding is a **provider/encounter mismatch**.

The medication review was recorded against:

```text
Encounter 161653
Provider 3308
Charlene Shookoff M.D.
Reviewed SNOMED: 428191000124101
```

But the MIPS report is for:

```text
Provider 17
Robert Schramm M.D.
Measure 130 / CMS68v6
```

The corresponding Robert Schramm encounter is:

```text
Encounter 161660
Provider 17
Medication attestation: NULL
```

Therefore, if CMS68 evaluates the encounter associated with the MIPS provider, the result will be:

```text
Robert's encounter
    ↓
No medication attestation
    ↓
Not Reviewed
    ↓
Not Met
```

while the chart can still display:

```text
Charlene's encounter
    ↓
Reviewed SNOMED exists
    ↓
Reviewed
```

---

## Root cause status

### Confirmed

* Robert Schramm (`17`) is configured for MIPS 2026.
* Robert has Measure 130 / CMS68v6 configured.
* Charlene Shookoff (`3308`) is not configured for MIPS 2026.
* Charlene does not have Measure 130 mapped.
* The Reviewed SNOMED code is stored on Charlene's encounter.
* Robert's same-date encounter has no medication attestation.
* MIPS displays the patient as Not Reviewed.

### Not yet confirmed

The following still requires source-code/business-rule verification:

> **Should a medication review performed on an encounter belonging to another provider count toward Robert Schramm's CMS68 measure?**

Until this rule is confirmed, it is not appropriate to conclusively classify the issue as a calculation defect.

---

# Detailed Technical Findings

## Provider configuration comparison

### Robert Schramm – Provider 17

```text
MIPS 2026 configuration: YES
Measure 130:             YES
CMS:                     CMS68v6
```

### Charlene Shookoff – Provider 3308

```text
MIPS 2026 configuration: NO
Measure 130:             NO
```

This distinction is critical.

---

## Encounter comparison

```text
Patient: 8109
Chart:   8346
Date:    08/19/2026
```

| Encounter | Provider | Provider Name          | Attestation       | MIPS CMS68              |
| --------: | -------: | ---------------------- | ----------------- | ----------------------- |
|    161653 |     3308 | Charlene Shookoff M.D. | `428191000124101` | Provider not configured |
|    161660 |       17 | Robert Schramm M.D.    | NULL              | Provider configured     |

---

## UI vs MIPS discrepancy

### Patient chart

```text
Current Medications
    ↓
Yes / Reviewed
```

### MIPS report

```text
Provider: Robert Schramm
Measure: CMS68v6
    ↓
Patient
    ↓
Not Reviewed
    ↓
0/1
```

### Patient-level popup

```text
Encounter Date: 08/19/2026
Status: Not Reviewed
```

The discrepancy is therefore reproducible at the data level.

---

# Important Misleading Assumptions

## `current_medication_status` is not sufficient

The `current_medication` table contains:

```text
current_medication_status
```

with values:

```text
11, 14, 15, 16, 18, 20
```

However, no evidence was established that these values directly represent the MIPS medication-review attestation.

The investigation instead focused on:

```text
encounter.medication_attestation_status
```

because the application behavior indicates that the Reviewed SNOMED is stored there.

---

## Provider name must not be inferred using unsafe casting

The attempted join:

```sql
CAST(e.emp_profile_doctorid AS INTEGER)
```

failed because `emp_profile_doctorid` contains values such as:

```text
G235
```

Therefore, future queries should not assume that `emp_profile_doctorid` is numeric.

---

## `measure_details` is global, not provider-specific

A query that joins:

```sql
measure_details
ON measure_details.measure_id = '130'
```

can return:

```text
CMS68v6
Documentation of Current Medications...
```

even for a provider who does not have Measure 130 configured.

Therefore:

> `measure_details` proves that Measure 130 exists, but does **not** prove that the provider has the measure configured.

Provider-specific configuration must be established through:

```text
quality_measures_provider_mapping
```

and:

```text
macra_provider_configuration
```

---

# SQL Analysis and Scripts

## Investigation Queries

### 1. Inspect current medication status distribution

**Type:** Investigation query

```sql
SELECT 
    current_medication_status,
    COUNT(*) AS count
FROM current_medication
GROUP BY current_medication_status
ORDER BY current_medication_status;
```

### Purpose

Determines which `current_medication_status` values exist and their frequency.

### Result

```text
11 → 1
14 → 46324
15 → 53028
16 → 864
18 → 25
20 → 2414
```

### Conclusion

This query did not establish which status represents MIPS medication review.

---

## 2. Inspect medications for initial exploratory patient

**Type:** Investigation query

```sql
SELECT
    current_medication_id,
    current_medication_patient_id,
    current_medication_rx_name,
    current_medication_status,
    current_medication_is_active,
    current_medication_order_on,
    current_medication_modified_on,
    current_medication_modified_by
FROM current_medication
WHERE current_medication_patient_id = 17621
ORDER BY current_medication_order_on DESC;
```

### Purpose

Inspect medication records and their statuses for patient `17621`.

### Note

Patient `17621` was part of the initial exploratory database investigation. The final affected support patient was later identified as patient `8109`.

---

## 3. Check a specific medication status

**Type:** Investigation query

```sql
SELECT DISTINCT
    current_medication_status
FROM current_medication
WHERE current_medication_status = 15;
```

### Purpose

Verify existence of status `15`.

### Conclusion

Status `15` exists, but this query does not establish its business meaning.

---

## 4. Attempted provider/MIPS query

**Type:** Investigation query — failed

```sql
SELECT
    m.macra_provider_configuration_provider_id AS provider_id,
    e.emp_profile_fullname AS provider_name,
    m.macra_provider_configuration_reporting_year AS reporting_year,
    m.macra_provider_configuration_reporting_method AS reporting_method,
    m.macra_provider_configuration_report_type AS report_type
FROM macra_provider_configuration m
LEFT JOIN emp_profile e
    ON CAST(e.emp_profile_doctorid AS INTEGER) =
       m.macra_provider_configuration_provider_id
WHERE m.macra_provider_configuration_reporting_year = 2026
  AND m.macra_provider_configuration_provider_id IN (3308, 1442, 17, 1)
ORDER BY m.macra_provider_configuration_provider_id;
```

### Result

```text
ERROR: invalid input syntax for type integer: "G235"
```

### Cause

`emp_profile_doctorid` contains alphanumeric values.

### Recommendation

Do not use this cast unless the column is safely validated with a numeric predicate.

---

## 5. Check provider MIPS configuration directly

**Type:** Validation query

```sql
SELECT *
FROM macra_provider_configuration
WHERE macra_provider_configuration_reporting_year = 2026
  AND macra_provider_configuration_provider_id IN (3308, 1442, 17, 1)
ORDER BY macra_provider_configuration_provider_id;
```

### Result

2026 configuration existed for:

```text
Provider 1
Provider 17
Provider 1442
```

No 2026 configuration existed for:

```text
Provider 3308
```

---

## 6. Check all measures mapped to providers

**Type:** Investigation / validation query

```sql
SELECT
    q.quality_measures_provider_mapping_provider_id AS provider_id,
    q.quality_measures_provider_mapping_reporting_year AS reporting_year,
    q.quality_measures_provider_mapping_measure_id AS measure_id,
    q.quality_measures_provider_mapping_cmsid AS cms_id,
    q.quality_measures_provider_mapping_title AS title,
    q.quality_measures_provider_mapping_priority AS priority
FROM quality_measures_provider_mapping q
WHERE q.quality_measures_provider_mapping_reporting_year = 2026
  AND q.quality_measures_provider_mapping_provider_id IN (3308, 1442, 17, 1)
ORDER BY
    q.quality_measures_provider_mapping_provider_id,
    q.quality_measures_provider_mapping_measure_id;
```

### Important result

Provider `17` had:

```text
128
130
226
236
238
317
374
65
IA_AHE_6
IA_EPA_2
IA_PM_13
```

Provider `1442` had:

```text
1
128
130
134
226
236
317
318
IA_AHE_6
IA_EPA_2
IA_PM_13
```

Provider `1` had:

```text
1
128
130
134
226
236
317
318
65
IA_AHE_6
IA_EPA_2
IA_PM_13
```

Provider `3308` had no 2026 mappings.

---

## 7. Provider + MIPS configuration + measure query

**Type:** Validation query

```sql
SELECT
    p.provider_id,
    p.provider_name,
    CASE
        WHEN m.macra_provider_configuration_id IS NOT NULL
        THEN 'CONFIGURED'
        ELSE 'NOT CONFIGURED'
    END AS mips_2026_status,
    q.quality_measures_provider_mapping_measure_id AS measure_id,
    q.quality_measures_provider_mapping_cmsid AS cms_id,
    q.quality_measures_provider_mapping_title AS measure_title
FROM (
    VALUES
        (3308, 'Charlene Shookoff M.D'),
        (1442, 'Glenwood undefined'),
        (17,   'Robert Schramm M.D.'),
        (1,    'Test Doctor M.D.')
) AS p(provider_id, provider_name)
LEFT JOIN macra_provider_configuration m
    ON m.macra_provider_configuration_provider_id = p.provider_id
   AND m.macra_provider_configuration_reporting_year = 2026
LEFT JOIN quality_measures_provider_mapping q
    ON q.quality_measures_provider_mapping_provider_id = p.provider_id
   AND q.quality_measures_provider_mapping_reporting_year = 2026
ORDER BY p.provider_id, q.quality_measures_provider_mapping_measure_id;
```

### Result

Confirmed:

```text
1    → CONFIGURED
17   → CONFIGURED
1442 → CONFIGURED
3308 → NOT CONFIGURED
```

---

## 8. Provider-specific Measure 130 query

**Type:** Validation query

```sql
SELECT
    p.provider_id,
    p.provider_name,

    CASE
        WHEN m.macra_provider_configuration_id IS NOT NULL
        THEN 'YES'
        ELSE 'NO'
    END AS mips_2026_configured,

    CASE
        WHEN q.quality_measures_provider_mapping_id IS NOT NULL
        THEN 'YES'
        ELSE 'NO'
    END AS measure_130_configured,

    q.quality_measures_provider_mapping_measure_id AS measure_id,
    md.cms_id,
    md.title

FROM (
    VALUES
        (3308, 'Charlene Shookoff M.D'),
        (17,   'Robert Schramm M.D.')
) AS p(provider_id, provider_name)

LEFT JOIN macra_provider_configuration m
    ON m.macra_provider_configuration_provider_id = p.provider_id
    AND m.macra_provider_configuration_reporting_year = 2026

LEFT JOIN quality_measures_provider_mapping q
    ON q.quality_measures_provider_mapping_provider_id = p.provider_id
    AND q.quality_measures_provider_mapping_reporting_year = 2026
    AND q.quality_measures_provider_mapping_measure_id = '130'

LEFT JOIN measure_details md
    ON md.measure_id = '130'

ORDER BY p.provider_id;
```

### Result

```text
Provider 17
    MIPS 2026: YES
    Measure 130: YES
    CMS68v6

Provider 3308
    MIPS 2026: NO
    Measure 130: NO
```

This is one of the most important validation queries from the investigation.

---

## 9. Get all measures for providers 17 and 3308

**Type:** Validation query

```sql
SELECT
    q.quality_measures_provider_mapping_provider_id AS provider_id,
    q.quality_measures_provider_mapping_measure_id AS measure_id,
    md.cms_id,
    md.title
FROM quality_measures_provider_mapping q
LEFT JOIN measure_details md
    ON md.measure_id = q.quality_measures_provider_mapping_measure_id
WHERE q.quality_measures_provider_mapping_reporting_year = 2026
  AND q.quality_measures_provider_mapping_provider_id IN (17, 3308)
ORDER BY
    q.quality_measures_provider_mapping_provider_id,
    q.quality_measures_provider_mapping_measure_id;
```

### Result

Provider `17` has Measure `130`.

Provider `3308` has no mapping.

---

## 10. Inspect affected patient encounters

**Type:** Primary investigation query

```sql
SELECT
    encounter_id,
    encounter_chartid,
    encounter_date,
    encounter_type,
    encounter_status,
    encounter_service_doctor,
    encounter_med_review,
    medication_attestation_status,
    encounter_snomed_code,
    encounter_code,
    encounter_status,
    encounter_modifiedby,
    encounter_modifiedon
FROM encounter
WHERE encounter_chartid = 8346
  AND encounter_date::date = '2026-08-19';
```

### Result

```text
161660 | 8346 | 2026-08-19 16:42:11 | ... | provider 17   | attestation NULL
161653 | 8346 | 2026-08-19 16:14:40 | ... | provider 3308 | attestation 428191000124101
```

### Interpretation

```text
161653 → Charlene → Reviewed
161660 → Robert   → Not Reviewed
```

---

## 11. Inspect all 2026 encounters for affected chart

**Type:** Investigation query

```sql
SELECT
    e.encounter_id,
    e.encounter_chartid,
    e.encounter_date,
    e.encounter_service_doctor,
    e.encounter_med_review,
    e.medication_attestation_status,
    e.encounter_snomed_code
FROM encounter e
WHERE e.encounter_chartid = 8346
  AND e.encounter_date >= '2026-01-01'
  AND e.encounter_date < '2027-01-01'
ORDER BY e.encounter_date DESC;
```

### Result

```text
162312 → 2026-09-15 → provider blank → attestation blank
161660 → 2026-08-19 → provider 17    → attestation blank
161653 → 2026-08-19 → provider 3308  → 428191000124101
```

This confirmed that the two August 19 encounters belong to different providers.

---

## 12. Inspect specific affected encounters

**Type:** Validation query

```sql
SELECT
    e.encounter_id,
    e.encounter_chartid,
    e.encounter_date,
    e.encounter_type,
    e.encounter_status,
    e.encounter_service_doctor,
    e.encounter_med_review,
    e.medication_attestation_status,
    e.encounter_created_by,
    e.encounter_created_date,
    e.encounter_modifiedby,
    e.encounter_modifiedon
FROM encounter e
WHERE e.encounter_id IN (161653, 161660)
ORDER BY e.encounter_id;
```

### Result

#### Encounter 161653

```text
Provider                 : 3308
Provider                 : Charlene Shookoff M.D.
Medication attestation  : 428191000124101
Created by               : 3518
Modified by              : 3518
```

#### Encounter 161660

```text
Provider                 : 17
Provider                 : Robert Schramm M.D.
Medication attestation  : NULL
Created by               : 3016
Modified by              : 3016
```

---

# Database Schema Findings

## `encounter`

Relevant structure:

```text
encounter_id
encounter_chartid
encounter_date
encounter_service_doctor
encounter_med_review
medication_attestation_status
encounter_snomed_code
encounter_code
encounter_status
encounter_created_by
encounter_created_date
encounter_modifiedby
encounter_modifiedon
```

### Useful indexes

The table contains indexes on:

```text
encounter_encounter_chartid
encounter_encounter_date
encounter_service_doctor_idx
encounter_id_service_doctor_idx
encounter_snomed_code_idx
encounter_status_idx
medication_attestation_status_idx
```

This supports the main investigation filters reasonably well.

---

## `current_medication`

Relevant fields:

```text
current_medication_id
current_medication_encounter_id
current_medication_patient_id
current_medication_status
current_medication_is_active
current_medication_order_on
current_medication_modified_on
```

### Useful indexes

```text
current_medication_encounter_id_index
current_medication_patient_id_index
current_medication_status_index
current_medication_status_patient_idx
current_medication_is_active_idx
current_medication_order_on_idx
```

---

## `macra_provider_configuration`

Important constraint:

```text
UNIQUE(provider_id, reporting_year)
```

This is useful because a provider should have at most one configuration row for a reporting year under this constraint.

---

## `quality_measures_provider_mapping`

Only the primary key was observed:

```text
quality_measures_provider_mapping_pkey
```

No composite index was shown for:

```text
provider_id
reporting_year
measure_id
```

### Performance consideration

If this table becomes large, queries filtering by:

```sql
provider_id
reporting_year
measure_id
```

may benefit from an appropriate composite index.

Any index change should be validated using:

```sql
EXPLAIN ANALYZE
```

against production-like data before deployment.

---

## `measure_details`

The observed indexes did not include an index specifically on:

```text
measure_id
```

If `measure_details` is large and frequently joined by `measure_id`, query performance should be monitored.

No index modification was made during this investigation.

---

# SQL Fix / Migration Status

## No database fix was performed

No:

* `UPDATE`
* `DELETE`
* `INSERT`
* Flyway migration
* stored procedure change

was performed as part of this investigation.

This is intentional.

### Reason

The existing data is valuable for reproducing the discrepancy:

```text
Charlene encounter → Reviewed
Robert encounter   → Not Reviewed
```

Manually updating the Robert encounter could hide the underlying calculation/provider-association problem.

---

# Code Changes

## Current status

**No code change was identified or implemented in this investigation.**

The next code investigation should focus on the Measure 130/CMS68 calculation and the patient-level Not Met detail logic.

---

# Code Investigation Required

Search the application source for:

```text
130
CMS68
CMS68v6
medication_attestation_status
428191000124101
Documentation of Current Medications in the Medical Record
```

Also inspect code containing:

```text
encounter_service_doctor
encounter_chartid
encounter_id
patientId
chartId
```

### Specifically trace

```text
MIPS report
    ↓
Measure 130 calculation
    ↓
Patient qualifying encounter selection
    ↓
Medication attestation lookup
    ↓
Reviewed / Not Reviewed
    ↓
Patient detail popup
```

The exact query/method responsible for the popup should be identified.

---

# Root Cause Decision Tree

The final RCA depends on the intended CMS68 business rule.

## Case 1 – Provider-specific calculation is intended

If the rule is:

> CMS68 evaluates medication review only on an encounter belonging to the MIPS-configured provider.

Then the current result is expected:

```text
Robert / 17
    ↓
CMS68 configured
    ↓
Encounter 161660
    ↓
No attestation
    ↓
Not Reviewed
```

Charlene's review does not count because:

```text
Charlene / 3308
    ↓
Not MIPS configured
    ↓
No CMS68 mapping
```

In this case, this is primarily a **provider/encounter workflow misunderstanding**, not necessarily a software defect.

---

## Case 2 – Patient-level medication review should count across providers

If the business rule is:

> A valid medication review for the patient should satisfy CMS68 regardless of which provider's encounter recorded the review.

Then the current behavior indicates a **calculation/filtering defect**.

The MIPS calculation would need to recognize:

```text
Encounter 161653
Provider 3308
Reviewed SNOMED
```

when evaluating the patient's CMS68 status for Robert.

This has not yet been confirmed.

---

# Fixes and Workarounds

## Permanent fix

**Not yet determined.**

A permanent fix must be based on confirmation of the intended CMS68 provider/encounter rule.

---

## Temporary operational workaround

If the intended workflow requires the MIPS-configured provider to perform the medication review, the medication review should be completed against the appropriate encounter for:

```text
Robert Schramm M.D.
Provider ID: 17
```

However, this should only be treated as a workaround after confirming with the product/business team that provider-specific review is the intended behavior.

---

## Do not manually modify the database

Avoid manually setting:

```sql
encounter.medication_attestation_status
```

or:

```sql
encounter.encounter_med_review
```

without confirming the application's normal workflow.

Manual modification can:

* hide the underlying defect,
* produce inconsistent audit information,
* affect future MIPS calculations,
* make reproduction more difficult.

---

# Validation and Testing

## Database validation performed

The following were verified:

* Patient ID and chart ID.
* Affected encounter date.
* Encounter provider IDs.
* Medication attestation status.
* 2026 MIPS provider configuration.
* Provider Measure 130 mapping.
* CMS68v6 measure definition.

---

## UI validation performed

The screenshots demonstrated:

### Patient chart

```text
Current Medications
→ Yes
→ Reviewed
```

### MIPS report

```text
Provider: Robert Schramm M.D.
Measure: Documentation of Current Medications in the Medical Record
Patient: GREYSON ADAMS
Performance Ratio: 0/1
```

### Patient-level detail

```text
Encounter Date: 08/19/2026
Status: Not Reviewed
```

---

# Recommended Test Scenarios

Once the calculation logic is identified or changed, test at least these scenarios.

## Test 1 – MIPS provider reviews medication

```text
Provider 17
Measure 130 configured
Robert's encounter
Reviewed SNOMED present
```

Expected:

```text
MIPS = Met
```

---

## Test 2 – MIPS provider does not review medication

```text
Provider 17
Measure 130 configured
Robert's encounter
No attestation
```

Expected:

```text
MIPS = Not Met
```

---

## Test 3 – Non-MIPS provider reviews medication

```text
Provider 3308
MIPS not configured
Reviewed SNOMED present
```

Verify whether the review should count toward another provider's MIPS calculation.

This is the **critical business-rule test for the reported issue**.

---

## Test 4 – Two encounters on the same date

```text
Encounter A → Provider 3308 → Reviewed
Encounter B → Provider 17   → Not Reviewed
```

Verify which encounter CMS68 chooses.

This exactly reproduces the reported case.

---

## Test 5 – Multiple encounters for same patient

Verify whether CMS68 uses:

* latest encounter,
* latest qualifying encounter,
* provider-specific encounter,
* encounter associated with the MIPS provider,
* any encounter during the measurement period,
* or another selection rule.

---

## Test 6 – Recalculation

After performing a valid review:

1. Recalculate MIPS.
2. Check overall percentage.
3. Check patient numerator/denominator.
4. Open the Not Met list.
5. Open patient-level details.
6. Confirm the patient is removed from Not Met if expected.

This is important because the reported issue involves a potential difference between **overall calculation** and **patient-level detail**.

---

# Risks and Side Effects

## Data integrity risk

Do not manually copy the Reviewed SNOMED code from one provider's encounter to another without business confirmation.

The two encounters are separate database records.

---

## Provider attribution risk

Changing provider filtering could affect MIPS results for multiple providers.

For example, allowing a review by provider `3308` to satisfy provider `17` could potentially alter:

* numerator counts,
* performance rates,
* patient-level status,
* MIPS submission data.

---

## Reporting risk

CMS68 is a reporting measure. Any calculation change must be validated against existing patient data and other providers.

---

## Backward compatibility

Any change to encounter selection logic could affect historical reporting years.

The calculation should ensure that a 2026 fix does not unintentionally alter 2025 or earlier reporting.

---

# Performance Considerations

The current investigation queries are relatively targeted.

The main filters use indexed fields such as:

```text
encounter_chartid
encounter_date
encounter_service_doctor
current_medication_patient_id
current_medication_encounter_id
```

However:

```text
quality_measures_provider_mapping
```

does not show a composite index for:

```text
provider_id + reporting_year + measure_id
```

and:

```text
measure_details
```

does not show an explicit `measure_id` index.

Before adding indexes, use:

```sql
EXPLAIN ANALYZE
```

on the actual production-scale query.

No index changes were made during this investigation.

---

# APIs / Services / Background Jobs

No specific API endpoint, Java service, background job, or stored procedure was identified from the conversation as the definitive source of the CMS68 calculation.

The next investigation must identify:

1. MIPS Measure 130 calculation service.
2. Patient-level Not Met query/service.
3. Endpoint used by the **Patient's Current Medication Status** popup.
4. Query used to select the qualifying encounter.
5. Logic used to interpret `medication_attestation_status`.

Therefore:

> **API/service ownership remains an open investigation item.**

---

# Current Evidence Matrix

| Evidence                           | Finding                |
| ---------------------------------- | ---------------------- |
| Patient                            | GREYSON ADAMS          |
| Patient ID                         | 8109                   |
| Chart ID                           | 8346                   |
| Affected date                      | 08/19/2026             |
| MIPS provider                      | Robert Schramm M.D.    |
| MIPS provider ID                   | 17                     |
| MIPS 2026 configured               | Yes                    |
| Measure                            | 130                    |
| CMS                                | CMS68v6                |
| Measure configured for provider 17 | Yes                    |
| Other provider                     | Charlene Shookoff M.D. |
| Other provider ID                  | 3308                   |
| MIPS configured for provider 3308  | No                     |
| CMS68 configured for provider 3308 | No                     |
| Reviewed encounter                 | 161653                 |
| Reviewed provider                  | 3308                   |
| Reviewed SNOMED                    | 428191000124101        |
| Robert's encounter                 | 161660                 |
| Robert's attestation               | NULL                   |
| MIPS patient result                | Not Met                |
| Chart result                       | Reviewed               |
| Database modification              | None                   |
| Code modification                  | None                   |
| Final code-level RCA               | Pending                |

---

# Current RCA Statement

> **The affected patient's medication review is recorded as a Reviewed SNOMED (`428191000124101`) on encounter `161653`, which belongs to Charlene Shookoff M.D. (Provider ID 3308). Charlene is not configured for 2026 MIPS and does not have Measure 130/CMS68v6 configured. The MIPS report is being evaluated for Robert Schramm M.D. (Provider ID 17), who is configured for 2026 MIPS and Measure 130. Robert's separate encounter `161660` on the same date does not contain a medication attestation. Consequently, the MIPS patient-level calculation reports the patient as Not Reviewed. The remaining question is whether CMS68 is intentionally provider/encounter-specific or whether a medication review performed by another provider should satisfy the MIPS measure. This requires confirmation from the measure business logic and source-code implementation.**

---

# Support Email Response

The following response was prepared based on the confirmed database findings:

> **Dear Idhayavani,**
>
> We checked the reported MIPS medication review issue for the affected patient.
>
> The medication was marked as **Reviewed** in the patient chart under the encounter serviced by **Charlene Shookoff M.D.** However, Charlene Shookoff M.D. is **not configured for MIPS for the 2026 reporting year** and does not have the **Documentation of Current Medications in the Medical Record (CMS68v6)** measure configured.
>
> The MIPS report is being evaluated for **Robert Schramm M.D.**, who is configured for MIPS 2026 and has the **Documentation of Current Medications in the Medical Record (CMS68v6)** measure configured. The corresponding Robert Schramm encounter does not have the medication review attestation recorded.
>
> Therefore, the medication review performed under Charlene Shookoff's encounter is not being counted toward Robert Schramm's MIPS **Documentation of Current Medications in the Medical Record** measure, which is why the patient continues to appear as **Not Met** in the MIPS report.
>
> Kindly check and let us know if you have any concerns.
>
> **Regards,**
> Gobala Krishnan

---

# Pending Work

## 1. Confirm CMS68 business rule

Determine whether medication review is:

* provider-specific,
* encounter-specific,
* patient-specific,
* or based on the latest qualifying encounter.

---

## 2. Trace Measure 130 source code

Find the exact Java/service/query responsible for:

```text
Measure 130
CMS68v6
Medication Review
```

---

## 3. Trace patient-level Not Met popup

Identify the code that produces:

```text
Patient's Current Medication Status
→ Not Reviewed
```

Determine whether it uses the same calculation logic as the main MIPS report.

---

## 4. Investigate the percentage increase

The customer reported that the overall percentage increased after medication review, while the patient remained in Not Met.

This should be reproduced to determine whether:

```text
overall numerator calculation
```

and:

```text
patient-level status calculation
```

use different data sources or encounter-selection logic.

---

## 5. Validate provider attribution

The UI shows:

```text
Charlene Shookoff M.D.
```

while the MIPS report is:

```text
Robert Schramm M.D.
```

The reason for having two encounters on the same date should be understood as part of the normal application workflow.

---

# Lessons Learned

## 1. Always verify provider configuration before debugging a MIPS measure

For a MIPS issue, establish:

```text
Provider
    ↓
MIPS reporting-year configuration
    ↓
Measure mapping
    ↓
Patient encounter
    ↓
Clinical data
```

Do not assume that because a measure exists globally, it is configured for the provider.

---

## 2. Distinguish measure definition from provider mapping

`measure_details` answers:

> Does Measure 130/CMS68 exist?

`quality_measures_provider_mapping` answers:

> Is Measure 130 configured for this provider?

`macra_provider_configuration` answers:

> Is the provider configured for MIPS for this reporting year?

These are separate concepts.

---

## 3. Provider mismatch is a critical troubleshooting signal

When the chart says:

```text
Reviewed
```

but MIPS says:

```text
Not Reviewed
```

check:

```text
encounter_service_doctor
```

before assuming the clinical data was lost.

In this case, the provider mismatch was the most significant discovery.

---

## 4. Check encounter-level data before changing medication records

Medication data can be associated with specific encounters.

Always inspect:

```text
current_medication_encounter_id
```

and:

```text
encounter_service_doctor
```

before modifying medication data.

---

## 5. Do not manually update production data while RCA is incomplete

The current records provide an excellent reproduction:

```text
Reviewed on provider A's encounter
Not Reviewed on provider B's encounter
```

Changing the data prematurely could destroy the evidence required to identify the actual calculation behavior.

---

## 6. MIPS overall percentage and patient-level status should be validated independently

A percentage increase does not necessarily prove that every individual patient has been correctly removed from the Not Met list.

For MIPS defects, validate both:

```text
Aggregate:
Numerator / Denominator / Performance Rate
```

and:

```text
Patient-level:
Met / Not Met / Exclusion / Exception
```

---
