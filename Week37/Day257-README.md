# MIPS – Pencil Icon Not Showing for All Eligible Encounters

**Case Reference Template**

---

## 1. Case Overview

| Field               | Value                            |
| ------------------- | -------------------------------- |
| **Module**          | MIPS                             |
| **Account**         | California Huntington Physicians |
| **Reported By**     | Dr. Joseph Nassir, M.D.          |
| **Assigned To**     | Gobala Krishnan                  |
| **Priority / Type** | Medium / Verify                  |
| **Status**          | Open                             |
| **Reporting Year**  | 2026                             |

---

## 2. Issue Description

**Problem:**
For **BP & Follow-up**, the **document (pencil) icon** in **MIPS Flowsheet** is visible for **only one encounter**, but **not for all eligible encounters**.

**Affected Measure:**

* **Measure ID:** 236
* **CMS ID:** CMS165v13
* **Measure Name:** Controlling High Blood Pressure

---

## 3. Environment / URLs

```text
New Encounter:
localhost:8080/GlaceDev/jsp/chart/patientdetails/NewEncounter.Action?Chart_Id=377445&patientId=377165&encounterType=1

MIPS Flowsheet:
localhost:8080/GlaceDev/jsp/chart/leafs/soap/MacraNewFlowsheet.jsp?patientId=377165&providerId=1379&encdate=2026-01-01&encounterId=154751
```

---

## 4. Patient & Encounter Details

| Field              | Value      |
| ------------------ | ---------- |
| **Patient ID**     | 377165     |
| **Chart ID**       | 377445     |
| **Encounter ID**   | 154751     |
| **Encounter Date** | 2026-01-01 |
| **Provider ID**    | 1379       |
| **Age in 2026**    | 23         |

---

## 5. Provider & Measure Configuration

### 5.1 Provider Validation

```sql
SELECT emp_profile_empid, emp_profile_fullname, emp_profile_is_active
FROM emp_profile
WHERE emp_profile_empid = 1379;
```

✅ Provider is **active**

---

### 5.2 Provider Configuration (Reporting Year)

```sql
SELECT *
FROM macra_provider_configuration
WHERE macra_provider_configuration_provider_id = 1379
  AND macra_provider_configuration_reporting_year = 2026;
```

---

### 5.3 Measure Mapping Validation

```sql
SELECT
  ep.emp_profile_fullname AS provider_name,
  mpc.macra_provider_configuration_reporting_year AS year,
  qmpm.quality_measures_provider_mapping_measure_id AS measure_id,
  md.title AS measure_name
FROM macra_provider_configuration mpc
JOIN quality_measures_provider_mapping qmpm
  ON qmpm.quality_measures_provider_mapping_provider_id =
     mpc.macra_provider_configuration_provider_id
 AND qmpm.quality_measures_provider_mapping_reporting_year =
     mpc.macra_provider_configuration_reporting_year
LEFT JOIN measure_details md
  ON md.measure_id = qmpm.quality_measures_provider_mapping_measure_id
LEFT JOIN emp_profile ep
  ON ep.emp_profile_empid = mpc.macra_provider_configuration_provider_id
WHERE mpc.macra_provider_configuration_provider_id = 1379
  AND mpc.macra_provider_configuration_reporting_year = 2026;
```

**Mapped Measures (Final):**

* 66 – Appropriate Testing for Children with Pharyngitis
* 317 – Screening for High BP & Follow-Up
* 236 – Controlling High Blood Pressure

---

### 5.4 Insert Measure Mapping (If Missing)

```sql
INSERT INTO quality_measures_provider_mapping (
  quality_measures_provider_mapping_provider_id,
  quality_measures_provider_mapping_reporting_year,
  quality_measures_provider_mapping_measure_id
) VALUES (1379, 2026, '236');
```

---

## 6. Patient Eligibility Checks

### 6.1 Age Validation

```sql
SELECT patient_registration_id,
       DATE_PART('year', AGE('2026-12-31', patient_registration_dob)) AS age
FROM patient_registration
WHERE patient_registration_id = 377165;
```

✅ Age ≥ 18 → Eligible

---

## 7. Encounter & CPT Validation

### 7.1 Encounter

```sql
SELECT encounter_id, encounter_date, encounter_status
FROM encounter
WHERE encounter_id = 154751;
```

---

### 7.2 CPT Validation

```sql
SELECT sd.service_detail_id,
       sd.service_detail_dos,
       c.cpt_cptcode
FROM service_detail sd
JOIN cpt c ON c.cpt_id = sd.service_detail_cptid
WHERE sd.service_detail_patientid = 377165
  AND sd.service_detail_sdoctorid = 1379
  AND sd.service_detail_dos BETWEEN '2026-01-01' AND '2026-12-31'
  AND c.cpt_cptcode IN ('99212','99213','99214','99215');
```

✅ CPT **99213** is eligible

---

## 8. BP Clinical Data Validation

```sql
SELECT
  ce.clinical_elements_name,
  pce.patient_clinical_elements_value,
  pce.patient_clinical_elements_created_on::date
FROM patient_clinical_elements pce
JOIN clinical_elements ce
  ON ce.clinical_elements_gwid = pce.patient_clinical_elements_gwid
WHERE pce.patient_clinical_elements_patientid = 377165
  AND pce.patient_clinical_elements_encounterid = 154751
  AND clinical_elements_name ILIKE '%BP%';
```

**Result:**

* Systolic BP = 118
* Diastolic BP = 76

✅ BP data present

---

## 9. Hypertension Diagnosis (Denominator Requirement)

### 9.1 Problem List / Assessment Check

```sql
WITH essential_htn AS (
  SELECT problem_list_patient_id AS patient_id, 'PROBLEM_LIST' AS source
  FROM problem_list
  WHERE problem_list_patient_id = 377165
    AND problem_list_dx_code = 'I10'
    AND problem_list_isactive = true
  UNION
  SELECT patient_assessments_patientid AS patient_id, 'ASSESSMENT' AS source
  FROM patient_assessments
  WHERE patient_assessments_patientid = 377165
    AND patient_assessments_dxcode = 'I10'
)
SELECT patient_id,
       COUNT(*) > 0 AS has_essential_hypertension,
       ARRAY_AGG(DISTINCT source) AS diagnosis_source
FROM essential_htn
GROUP BY patient_id;
```

❌ **No active I10 diagnosis found**

---

## 10. Root Cause

> **Measure 236 (Controlling High Blood Pressure)** requires an **active hypertension diagnosis (ICD-10 I10)** in **Problem List or Assessment**.
>
> Although BP readings, eligible CPT, age, and provider mapping are valid, the **missing I10 diagnosis prevents the encounter from qualifying**, so the **pencil icon does not appear for all encounters**.

---

## 11. Resolution / Recommendation

### Clinical Action Required

* Add **I10 – Essential Hypertension** to:

  * **Problem List (Active)** OR
  * **Assessment** for an eligible encounter

### Post-Fix

* Reload MIPS Flowsheet
* Pencil icon will appear for all eligible encounters

---

## 12. Final Status Summary

| Item                  | Status          |
| --------------------- | --------------- |
| Provider Mapping      | ✅ Verified      |
| CPT Eligibility       | ✅ Verified      |
| BP Data               | ✅ Present       |
| Diagnosis Requirement | ❌ Missing       |
| Root Cause            | Identified      |
| Fix Type              | Clinical / Data |
