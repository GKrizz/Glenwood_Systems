
# Day 354 – Case #245445

## PHQ-9 Not Showing as Completed – MIPS / sbaskaran

**Provider:** Sunny Baskaran, M.D.
**User:** sbaskaran
**Issue Area:** MACRA / MIPS – Depression Screening & Follow-Up
**Measure:** Preventive Care and Screening: Screening for Depression and Follow-Up Plan

---

# Issue Summary

Even after correcting:

* `risk_assessment_result_code_system`
* `risk_assessment_screening_id`

some patients were still appearing under:

```text id="t5x1oa"
NOT MET
```

in the MIPS/MACRA dashboard.

---

# Current Patient Status

| Account No | Screening Result | Follow-Up Present | Current Status |
| ---------- | ---------------- | ----------------- | -------------- |
| 030618     | Negative         | Not Required      | MET            |
| 030926     | Positive         | Missing           | NOT MET        |
| 031261     | Positive         | Missing           | NOT MET        |

---

# Root Cause

For MIPS Measure 134:

## Logic

| PHQ Result | Requirement                         |
| ---------- | ----------------------------------- |
| Negative   | Screening alone satisfies numerator |
| Positive   | Requires documented follow-up plan  |

The following patients had:

* Positive PHQ-9 result
* No follow-up documentation

Therefore the numerator failed correctly.

---

# Additional UI Issue

## Reported Problem

```text id="b0f28e"
Does not give a follow-up option
(the box they are supposed to click when positive)
```

This indicates the follow-up clinical elements were likely:

* Not configured
* Missing from Plan tab
* Not mapped to depression follow-up SNOMED codes

---

# Action Requested

```text id="mxs5o3"
@Kalaiselvan,
Please check the required elements are configured
in Plan tab for depression.
```

---

# Required Follow-Up Configuration

The Plan tab must contain properly mapped clinical elements for:

* Depression follow-up plan
* Referral
* Counseling
* Behavioral health referral
* Suicide risk assessment
* Medication management

These elements must map to the expected SNOMED concepts used by MIPS numerator logic.

---

# Investigation Query

## Validate PHQ Screening Records

```sql id="cm88ng"
SELECT
    ra.risk_assessment_id,
    pr.patient_registration_accountno,
    ra.risk_assessment_code,
    ra.risk_assessment_result_code,
    ra.risk_assessment_result_code_system,
    ra.risk_assessment_result_description,
    CASE
        WHEN ra.risk_assessment_result_code = '428171000124102'
            THEN 'Negative'
        WHEN ra.risk_assessment_result_code = '428181000124104'
            THEN 'Positive'
        ELSE ' '
    END AS result,
    ra.risk_assessment_modified_on
FROM risk_assessment ra
JOIN patient_registration pr
    ON pr.patient_registration_id = ra.risk_assessment_patient_id
WHERE pr.patient_registration_accountno IN ('011856')
  AND ra.risk_assessment_code IN ('73831-0','73832-8');
```

---

# PHQ Measure Logic

## Valid LOINC Codes

| Screening        | LOINC   |
| ---------------- | ------- |
| PHQ-9 Adult      | 73832-8 |
| PHQ-9 Adolescent | 73831-0 |

---

## Valid SNOMED Result Codes

| Result   | SNOMED          |
| -------- | --------------- |
| Negative | 428171000124102 |
| Positive | 428181000124104 |

---

# Follow-Up Requirement

For positive screenings:

```text id="hh0y1g"
Follow-up plan documentation is mandatory
```

Examples:

* Referral to behavioral health
* Counseling
* Medication treatment
* Safety planning
* Additional evaluation

Without follow-up documentation:

```text id="6kjm6g"
Numerator = NOT MET
```

---

# Additional Issue – CPT II / G-Code Logic

## Requested CPT II Codes

### HbA1c

| Condition | CPT II |
| --------- | ------ |
| < 7.0%    | 3044F  |
| 7.0–8.0%  | 3051F  |
| 8.0–9.0%  | 3052F  |
| > 9.0%    | 3046F  |

---

### Systolic BP

| Condition | CPT II |
| --------- | ------ |
| < 130     | 3074F  |
| 130–139   | 3075F  |
| >= 140    | 3077F  |

---

### Diastolic BP

| Condition | CPT II |
| --------- | ------ |
| < 80      | 3078F  |
| 80–89     | 3079F  |
| >= 90     | 3080F  |

---

# Existing Implementation Concern

## Observation

```text id="b7fd2m"
Mounika wrote this exact F-code logic into
ChargesServicesImpl.java
```

Later modified to:

```text id="ulc6me"
M1371, M1372 ...
```

which are:

* G-codes / M-codes
* globally hardcoded

---

# Suspected Root Cause

The logic inside:

```text id="yqqw7d"
ChargesServicesImpl.java
```

appears to use:

```text id="xgc0yk"
single hardcoded global code mapping
```

instead of:

* practice-specific configuration
* payer-specific logic
* CPT-II selection rules

---

# Impact

This may cause:

| Issue                          | Impact                       |
| ------------------------------ | ---------------------------- |
| CPT-II replaced with G/M-codes | Incorrect quality submission |
| Hardcoded mappings             | All practices affected       |
| No configuration layer         | Cannot customize payer logic |
| Global implementation          | Incorrect code generation    |

---

# Recommended Investigation

## Review Logic In

```text id="4yq9bi"
ChargesServicesImpl.java
```

Check for:

* Hardcoded M1371/M1372 logic
* Replaced CPT-II mappings
* Global code overrides
* Practice-independent behavior

---

# Recommended Fix

## 1. Restore CPT-II Logic

Reintroduce:

* 3044F
* 3051F
* 3052F
* 3046F
* 3074F
* 3075F
* 3077F
* 3078F
* 3079F
* 3080F

based on actual measure values.

---

# 2. Make Logic Configurable

Instead of:

```java id="cw6vzm"
if (...) return "M1371";
```

use:

```java id="mny8ya"
MeasureCodeConfiguration
```

with:

* practice-level mapping
* payer-level mapping
* CMS version support

---

# 3. Add Follow-Up Plan Configuration Validation

Validate:

* Plan tab elements exist
* SNOMED mappings are correct
* Positive PHQ triggers follow-up section
* Follow-up checkbox renders properly

---
