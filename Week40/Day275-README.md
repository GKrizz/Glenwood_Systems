
# 📘 CMS2v14 — Preventive Care and Screening: Depression Screening & Follow-Up Plan

## SQL Logic Reference (GlaceEMR)

This document explains the **complete SQL logic** for calculating:

* ✅ Initial Population (IPP)
* ✅ Denominator Exceptions
* ✅ Numerator (Screening + Follow-up Plan)

for **CMS2v14 (Measure ID: 134)**.

---

# 🧠 Measure Logic — Simple Explanation

A patient qualifies if:

## ✅ Initial Population (IPP)

Patient must:

* Age **≥ 12 years** at start of Measurement Period
* Have **at least one qualifying encounter** during the Measurement Period

Measurement Period Example:

```text
2026-01-01 → 2026-12-31
```

---

## ❌ Denominator Exclusions / Exceptions

Exclude if patient has:

* **Bipolar disorder**
* **Depression diagnosis prior to encounter**

(ICD Codes: F30, F31 series)

---

## ✅ Numerator Requirements

Patient must have:

### 1️⃣ Depression Screening

* Valid screening tool recorded
* Within **14 days before or on encounter date**

LOINC Codes:

```text
73831-0 — PHQ-2
73832-8 — PHQ-9
```

---

### 2️⃣ Follow-Up Plan (if screening positive)

If screening result = **Positive**, must document:

* Referral OR
* Treatment plan OR
* Education

Example SNOMED:

```text
61801003 — Referral for Depression Adult
```

---

# 🗂️ STEP-BY-STEP SQL LOGIC

---

# STEP 1 — Age Eligibility

```sql
SELECT
    pr.patient_registration_id AS patient_id,
    pr.patient_registration_dob,
    EXTRACT(YEAR FROM AGE(DATE '2026-01-01', pr.patient_registration_dob)) AS age,
    CASE
        WHEN EXTRACT(YEAR FROM AGE(DATE '2026-01-01', pr.patient_registration_dob)) >= 12
        THEN 'AGE PASS'
        ELSE 'AGE FAIL'
    END AS age_status
FROM patient_registration pr
WHERE pr.patient_registration_id = :patient_id;
```

---

# STEP 2 — Qualifying Encounters

```sql
SELECT
    sd.service_detail_id,
    sd.service_detail_dos AS encounter_date,
    c.cpt_cptcode
FROM service_detail sd
JOIN cpt c ON c.cpt_id = sd.service_detail_cptid
WHERE sd.service_detail_patientid = :patient_id
AND sd.service_detail_dos BETWEEN '2026-01-01' AND '2026-12-31'
AND c.cpt_cptcode IN (
    '99201','99202','99203','99204','99205',
    '99212','99213','99214','99215',
    '99384','99385','99386','99387',
    '99394','99395','99396','99397',
    '90791','90792','90832','90834','90837'
);
```

---

# STEP 3 — Denominator Exceptions Check

## Bipolar / Depression Diagnosis Before Encounter

```sql
WITH qualifying_encounter AS (
    SELECT MIN(service_detail_dos::date) AS encounter_date
    FROM service_detail
    WHERE service_detail_patientid = :patient_id
)

SELECT *
FROM (
    SELECT patient_assessments_patientid AS patient_id,
           patient_assessments_encounterdate::date AS dx_date
    FROM patient_assessments
    WHERE patient_assessments_dxcode ~ '^F3[01]'

    UNION ALL

    SELECT problem_list_patient_id,
           COALESCE(problem_list_onset_date::date,
                    problem_list_createdon::date)
    FROM problem_list
    WHERE problem_list_dx_code ~ '^F3[01]'
) dx
JOIN qualifying_encounter qe
ON dx.dx_date < qe.encounter_date;
```

If rows exist → **Excluded from denominator**

---

# STEP 4 — Depression Screening Check

```sql
SELECT
    ra.risk_assessment_date AS screening_date,
    ra.risk_assessment_code AS loinc_code,
    ra.risk_assessment_result_code AS result_code
FROM risk_assessment ra
WHERE ra.risk_assessment_patient_id = :patient_id
AND ra.risk_assessment_code IN ('73831-0','73832-8')
ORDER BY ra.risk_assessment_date DESC;
```

---

## Valid Screening Window

```sql
SELECT *
FROM risk_assessment
WHERE risk_assessment_date BETWEEN
      encounter_date - INTERVAL '14 days'
  AND encounter_date;
```

---

# STEP 5 — Follow-Up Plan Check

Follow-up plan documented via:

```sql
SELECT
    ce.clinical_elements_name,
    ce.clinical_elements_snomed,
    pce.patient_clinical_elements_value
FROM patient_clinical_elements pce
JOIN clinical_elements ce
ON ce.clinical_elements_gwid = pce.patient_clinical_elements_gwid
WHERE pce.patient_clinical_elements_patientid = :patient_id
AND pce.patient_clinical_elements_encounterid = :encounter_id;
```

Example valid entry:

```text
Referral for Depression Adult
SNOMED = 61801003
```

---

# 🐛 Root Cause of Issue (Important)

## ❗ Problem

Numerator was NOT firing because:

```text
Depression screening result code system OID was missing
```

Without OID:

* Value-set matching fails
* Result cannot be mapped to SNOMED

---

## ✅ Fix Implemented

Fallback logic added:

```text
IF OID IS NULL
→ Default to SNOMED OID
2.16.840.1.113883.6.96
```

This allows:

* Correct value-set validation
* Proper numerator evaluation

---

# 📊 Example Patient Result

Patient ID: **377165**

```text
Age: 22 → PASS
Encounter: Present → PASS
Screening: Positive PHQ-9 → PASS
Follow-up: Referral documented → PASS

FINAL STATUS = NUMERATOR MET
```

---

# ⚠️ Common Implementation Mistakes

### ❌ Missing SNOMED OID

Always default when null.

---

### ❌ Ignoring 14-day screening window

Screening must be:

```text
Within 14 days before encounter
```

---

### ❌ Not checking follow-up when result = positive

Follow-up required ONLY for positive results.

---

# 🧩 Measure Flow Diagram

```text
AGE ≥ 12
     ↓
Qualifying Encounter
     ↓
Depression Screening Done?
     ↓
Result Positive?
     ↓
Follow-up Plan Documented?
     ↓
NUMERATOR MET
```

---
