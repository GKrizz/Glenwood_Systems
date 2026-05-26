
# Day 351 – Case #245445

## PHQ-9 Not Showing as Completed – MIPS

**Provider:** Sunny Baskaran, M.D.
**Measure:** Preventive Care and Screening: Screening for Depression and Follow-Up Plan
**MIPS Measure ID:** 134

---

# Issue Summary

PHQ-9 screenings were completed for multiple patients, but MIPS was not recognizing the screenings as completed.

Affected Accounts:

* 031261
* 030926
* 031065
* 031098
* 031243

Investigation revealed that the issue was caused by missing values in:

```sql
risk_assessment_result_code_system
```

The expected SNOMED code system OID should be:

```text
2.16.840.1.113883.6.96
```

Because this field was blank/null, the numerator logic for MIPS Measure 134 failed to identify the PHQ-9 screenings correctly.

---

# Affected Patient Mapping

```sql
SELECT
    patient_registration_id,
    patient_registration_accountno
FROM patient_registration
WHERE patient_registration_accountno IN (
    '031261',
    '030926',
    '031065',
    '031098',
    '031243'
);
```

Result:

| Patient ID | Account No | Expected Result |
| ---------- | ---------- | --------------- |
| 31261      | 031261     | Positive        |
| 30926      | 030926     | Positive        |
| 31065      | 031065     | Negative        |
| 31098      | 031098     | Negative        |
| 31243      | 031243     | Negative        |

---

# Investigation Steps

## 1. Verified PHQ Risk Assessment Data

```sql
SELECT
    risk_assessment_id,
    risk_assessment_patient_id,
    risk_assessment_result_code,
    risk_assessment_result_description,
    risk_assessment_code,
    risk_assessment_code_system,
    risk_assessment_result_code_system,
    risk_assessment_result_code_system_name,
    risk_assessment_performed_by,
    risk_assessment_performed_on
FROM risk_assessment
WHERE risk_assessment_patient_id = 31098
ORDER BY risk_assessment_performed_on DESC;
```

Observed:

```text
risk_assessment_result_code_system = NULL
```

---

# PHQ-9 Mapping Logic

## Valid LOINC Codes

| Screening Type | LOINC   |
| -------------- | ------- |
| PHQ-9 Adult    | 73831-0 |
| PHQ-9 Modified | 73832-8 |

---

## Valid SNOMED Result Codes

| Result   | SNOMED Code     |
| -------- | --------------- |
| Negative | 428171000124102 |
| Positive | 428181000124104 |

---

# Root Cause

The PHQ records existed correctly in `risk_assessment`, but the field below was empty:

```sql
risk_assessment_result_code_system
```

Expected Value:

```text
2.16.840.1.113883.6.96
```

Without the SNOMED OID, the MIPS numerator logic could not validate the screening outcome.

---

# Sample Problematic Records

```sql
SELECT
    risk_assessment_id,
    risk_assessment_patient_id,
    risk_assessment_result_code,
    risk_assessment_result_description,
    risk_assessment_code,
    risk_assessment_result_code_system
FROM risk_assessment
WHERE risk_assessment_patient_id = 31098
ORDER BY risk_assessment_performed_on DESC;
```

Output:

| risk_assessment_id | result_code     | description | loinc   | result_code_system |
| ------------------ | --------------- | ----------- | ------- | ------------------ |
| 5425               | 428171000124102 | Positive    | 73833-0 | NULL               |
| 5426               | 428171000124102 | Negative    | 73832-8 | NULL               |

---

# Validation Queries

## Find Invalid PHQ Records

```sql
SELECT
    ra.risk_assessment_id,
    pr.patient_registration_accountno,
    ra.risk_assessment_code,
    ra.risk_assessment_result_code,
    ra.risk_assessment_result_code_system,
    ra.risk_assessment_result_description
FROM risk_assessment ra
JOIN patient_registration pr
    ON pr.patient_registration_id = ra.risk_assessment_patient_id
WHERE ra.risk_assessment_code IN ('73831-0','73832-8')
  AND ra.risk_assessment_result_code IN (
        '428171000124102',
        '428181000124104'
      )
  AND (
        ra.risk_assessment_result_code_system IS NULL
        OR TRIM(ra.risk_assessment_result_code_system) = ''
      );
```

---

# Backup Procedure

## Backup Single Record

```sql
\copy (
    SELECT *
    FROM risk_assessment
    WHERE risk_assessment_id = 5426
) TO 'phq9_backup_031098_may_04_2026.csv'
WITH CSV HEADER;
```

---

## Backup All Affected Records

```sql
\copy (
    SELECT *
    FROM risk_assessment
    WHERE risk_assessment_code IN ('73831-0','73832-8')
      AND risk_assessment_result_code IN (
            '428171000124102',
            '428181000124104'
          )
      AND (
            risk_assessment_result_code_system IS NULL
            OR TRIM(risk_assessment_result_code_system) = ''
          )
) TO 'phq9_backup_all.csv'
WITH CSV HEADER;
```

---

# Fix Applied

## Single Record Update

```sql
UPDATE risk_assessment
SET risk_assessment_result_code_system = '2.16.840.1.113883.6.96'
WHERE risk_assessment_id = 5426;
```

---

## Bulk Safe Update

```sql
UPDATE risk_assessment
SET risk_assessment_result_code_system = '2.16.840.1.113883.6.96'
WHERE risk_assessment_code IN ('73831-0','73832-8')
  AND risk_assessment_result_code IN (
        '428171000124102',
        '428181000124104'
      )
  AND (
        risk_assessment_result_code_system IS NULL
        OR TRIM(risk_assessment_result_code_system) = ''
      );
```

---

# Post-Update Validation

## Verify Updated Records

```sql
SELECT
    risk_assessment_id,
    risk_assessment_result_code_system
FROM risk_assessment
WHERE risk_assessment_id = 5426;
```

Expected:

```text
2.16.840.1.113883.6.96
```

---

## Final Validation Count

```sql
SELECT COUNT(*)
FROM risk_assessment
WHERE risk_assessment_code IN ('73831-0','73832-8')
  AND risk_assessment_result_code IN (
        '428171000124102',
        '428181000124104'
      )
  AND (
        risk_assessment_result_code_system IS NULL
        OR TRIM(risk_assessment_result_code_system) = ''
      );
```

Expected Result:

```text
0
```

---

# Outcome

After updating the missing SNOMED OID values:

* PHQ-9 screenings were recognized correctly.
* MIPS Measure 134 numerator logic started passing.
* PHQ-9 status displayed properly as completed.
* Existing patient screenings became compliant without requiring re-entry.

---

# Conclusion

The issue was not related to missing PHQ-9 documentation or encounter eligibility.
The root cause was incomplete coding metadata in the `risk_assessment` table.

Specifically:

```text
risk_assessment_result_code_system
```

was blank/null for valid PHQ screening results.

Updating the value to:

```text
2.16.840.1.113883.6.96
```

resolved the issue successfully for all affected records.
