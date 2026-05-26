
# Day 353 – MACRA Tab `risk_assessment_screening_id` EMPTY Issue

## Issue Summary

The MACRA tab was not properly recognizing depression screening assessments because the field:

```text id="n9efr8"
risk_assessment_screening_id
```

was NULL for several PHQ-related records in the `risk_assessment` table.

This affected:

* PHQ-9 Adult
* PHQ-9 Adolescent
* PHQ-2 Adult
* Adult Depression Screening
* Adolescent Depression Screening

As a result:

* MACRA/MIPS screening logic failed
* Depression screenings were not mapped correctly
* Numerator calculations became inconsistent

---

# Patient Validation

## Patient Details

```sql id="8xllb4"
SELECT
  patient_registration_id,
  patient_registration_accountno,
  patient_registration_first_name,
  patient_registration_last_name,
  patient_registration_dob,
  EXTRACT(YEAR FROM AGE('2026-01-01', patient_registration_dob)) AS age_on_2026_01_01,
  patient_registration_sex,
  patient_registration_active
FROM patient_registration
WHERE patient_registration_id = 377165;
```

---

# Qualifying Encounter Validation

```sql id="wn0mzg"
SELECT
    sd.service_detail_id,
    sd.service_detail_patientid AS patient_id,
    c.cpt_cptcode AS cpt_code,
    sd.service_detail_dos AS visit_date
FROM service_detail sd
JOIN cpt c
    ON c.cpt_id = sd.service_detail_cptid
WHERE sd.service_detail_patientid = 377165
  AND sd.service_detail_dos BETWEEN '2026-01-01' AND '2026-12-31'
  AND c.cpt_cptcode IN (
    '99201','99202','99203','99204','99205',
    '99212','99213','99214','99215',
    '99384','99385','99386','99387',
    '99394','99395','99396','99397',
    '97003','96150','96151','96116','96118',
    '92625','90791','90792','90832','90834','90837'
  );
```

---

# Risk Assessment Investigation

## Full PHQ Screening Data

```sql id="4u9j66"
SELECT
     risk_assessment_id,
     risk_assessment_patient_id,
     risk_assessment_encounter_id,
     risk_assessment_screening_name,
     risk_assessment_result_code,
     risk_assessment_result_description,
     risk_assessment_code,
     risk_assessment_code_system,
     risk_assessment_result_code_system,
     risk_assessment_result_code_system_name,
     risk_assessment_performed_by,
     risk_assessment_performed_on,
     risk_assessment_date,
     risk_assessment_modified_on,
     risk_assessment_screening_id
FROM risk_assessment
WHERE risk_assessment_patient_id = 377165
ORDER BY risk_assessment_performed_on DESC;
```

---

# PHQ-Specific Validation

```sql id="v2jsli"
SELECT
     risk_assessment_id,
     risk_assessment_patient_id,
     risk_assessment_encounter_id,
     risk_assessment_screening_name,
     risk_assessment_result_code,
     risk_assessment_result_description,
     risk_assessment_code,
     risk_assessment_code_system,
     risk_assessment_result_code_system,
     risk_assessment_result_code_system_name,
     risk_assessment_performed_by,
     risk_assessment_performed_on,
     risk_assessment_date,
     risk_assessment_modified_on
FROM risk_assessment
WHERE risk_assessment_patient_id = 377165
  AND risk_assessment_code IN ('73831-0','73832-8')
ORDER BY risk_assessment_performed_on DESC;
```

---

# Root Cause

The primary issue was:

```text id="90y8aa"
risk_assessment_screening_id IS NULL
```

for multiple depression screening records.

The MACRA tab depends on valid screening identifiers to classify:

* PHQ-2
* PHQ-9
* Adult Depression Screening
* Adolescent Depression Screening

Without valid screening IDs:

* MACRA logic could not identify the screening type
* Numerator logic failed
* Screenings appeared incomplete

---

# Initial Direct Fix

## PHQ-9 Adult Mapping

```sql id="cjlwmz"
UPDATE risk_assessment
SET
    risk_assessment_screening_id = 1,
    risk_assessment_screening_name = 'PHQ-9 - Adult'
WHERE risk_assessment_code = '73832-8'
  AND risk_assessment_screening_id IS NULL;
```

---

## PHQ-9 Adolescent Mapping

```sql id="gcchom"
UPDATE risk_assessment
SET
    risk_assessment_screening_id = 2,
    risk_assessment_screening_name = 'PHQ-9 - Adolescent'
WHERE risk_assessment_code = '73831-0'
  AND risk_assessment_screening_id IS NULL;
```

---

# Additional Investigation

## Find All NULL Screening IDs

```sql id="6c75j9"
SELECT DISTINCT
    risk_assessment_description
FROM risk_assessment
WHERE risk_assessment_screening_id IS NULL;
```

Result:

| Description                     |
| ------------------------------- |
| Adult Depression Screening      |
| PHQ-9 - Adult                   |
| Adolescent Depression Screening |
| PHQ-2 - Adult                   |

---

# Total Impacted Records

```sql id="g8m4z8"
SELECT COUNT(*)
FROM risk_assessment
WHERE risk_assessment_screening_id IS NULL
  AND risk_assessment_description IN (
      'Adult Depression Screening',
      'Adolescent Depression Screening',
      'PHQ-9 - Adult',
      'PHQ-2 - Adult'
  );
```

Result:

```text id="mjlwmk"
4716
```

---

# Distribution Analysis

```sql id="ddn8fr"
SELECT
    risk_assessment_description,
    risk_assessment_screening_id,
    COUNT(*)
FROM risk_assessment
WHERE risk_assessment_description IN (
    'Adult Depression Screening',
    'Adolescent Depression Screening',
    'PHQ-9 - Adult',
    'PHQ-2 - Adult'
)
GROUP BY
    risk_assessment_description,
    risk_assessment_screening_id
ORDER BY
    risk_assessment_description,
    risk_assessment_screening_id;
```

## Observations

| Description                     | Screening ID | Count |
| ------------------------------- | ------------ | ----- |
| Adult Depression Screening      | NULL         | 3923  |
| PHQ-2 - Adult                   | NULL         | 539   |
| PHQ-9 - Adult                   | NULL         | 202   |
| Adolescent Depression Screening | NULL         | 52    |

Inconsistent mappings already existed:

| Description                | Existing Screening IDs |
| -------------------------- | ---------------------- |
| Adult Depression Screening | 1, 6                   |
| PHQ-9 - Adult              | 1, 6                   |
| PHQ-2 - Adult              | 1, 6                   |

This indicated historical data inconsistencies.

---

# Backup Procedure

## Backup All Affected Records

```sql id="ot1h5w"
\copy (
    SELECT *
    FROM risk_assessment
    WHERE risk_assessment_screening_id IS NULL
      AND risk_assessment_description IN (
          'Adult Depression Screening',
          'Adolescent Depression Screening',
          'PHQ-9 - Adult',
          'PHQ-2 - Adult'
      )
) TO 'risk_assessment_backup_before_screening_fix.csv'
WITH CSV HEADER;
```

---

# Final Fix Applied

## PHQ-2 Adult Mapping

```sql id="2pk6z4"
UPDATE risk_assessment
SET risk_assessment_screening_id = 6
WHERE risk_assessment_screening_id IS NULL
  AND risk_assessment_description = 'PHQ-2 - Adult';
```

---

## PHQ-9 / Adult Depression Mapping

```sql id="b2p4uk"
UPDATE risk_assessment
SET risk_assessment_screening_id = 1
WHERE risk_assessment_screening_id IS NULL
  AND risk_assessment_description IN (
      'Adult Depression Screening',
      'Adolescent Depression Screening',
      'PHQ-9 - Adult'
  );
```

---

# Post-Update Validation

## Verify Distribution

```sql id="wktkp1"
SELECT
    risk_assessment_description,
    risk_assessment_screening_id,
    COUNT(*)
FROM risk_assessment
WHERE risk_assessment_description IN (
    'Adult Depression Screening',
    'Adolescent Depression Screening',
    'PHQ-9 - Adult',
    'PHQ-2 - Adult'
)
GROUP BY
    risk_assessment_description,
    risk_assessment_screening_id
ORDER BY
    risk_assessment_description,
    risk_assessment_screening_id;
```

---

## Final NULL Validation

```sql id="tr2wz9"
SELECT COUNT(*)
FROM risk_assessment
WHERE risk_assessment_screening_id IS NULL
  AND risk_assessment_description IN (
      'Adult Depression Screening',
      'Adolescent Depression Screening',
      'PHQ-9 - Adult',
      'PHQ-2 - Adult'
  );
```

Expected Result:

```text id="e0a8x0"
0
```

---

# Key Findings

| Issue                               | Impact                             |
| ----------------------------------- | ---------------------------------- |
| NULL `risk_assessment_screening_id` | MACRA tab failed                   |
| Inconsistent screening mappings     | Numerator mismatch                 |
| Historical PHQ records incomplete   | MIPS logic impacted                |
| PHQ-2/PHQ-9 mixed IDs               | Incorrect screening categorization |

---

# Resolution

The issue was resolved by:

1. Identifying all NULL screening IDs
2. Backing up affected records
3. Standardizing screening mappings
4. Updating PHQ-related records with valid screening IDs
5. Re-validating all affected data

---
