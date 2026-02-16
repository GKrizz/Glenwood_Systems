# Case #241552 – Smoking Status entered in MIPS Flowsheet not saved – Dr. Awani Kumar (AKM)
---

## 👤 Patient Details

| Field          | Value      |
| -------------- | ---------- |
| Patient ID     | 17568      |
| Chart ID       | 17475      |
| Encounter ID   | 4209       |
| Provider ID    | 1          |
| Encounter Date | 2025-01-14 |

---

# 🧑‍⚕️ 1. Patient Demographics Verification

## 🎯 Purpose

To confirm:

* Patient age eligibility
* Active status
* Basic demographic details

---

## 🧾 SQL Query

```sql
SELECT
  patient_registration_id,
  patient_registration_accountno,
  patient_registration_first_name,
  patient_registration_last_name,
  patient_registration_dob,
  EXTRACT(YEAR FROM AGE('2025-01-01', patient_registration_dob)) AS age_on_2025_01_01,
  patient_registration_sex,
  patient_registration_active
FROM patient_registration
WHERE patient_registration_id = 17568;
```

---

# 🏥 2. Qualifying Encounter Validation

## 🎯 Purpose

To verify:

* Preventive visits
* Qualifying encounters
* Valid CPT codes within measurement period

---

## 🧾 SQL Query

```sql
SELECT
    sd.service_detail_id,
    sd.service_detail_patientid AS patient_id,
    c.cpt_cptcode AS cpt_code,
    sd.service_detail_dos AS visit_date,
    CASE
        WHEN c.cpt_cptcode IN (
            'G0438','G0439','99395','99396','99397',
            '99385','99386','99387',
            '99411','99412','99401','99402','99403','99404','99429'
        ) THEN 'PREVENTIVE_VISIT'
        ELSE 'QUALIFYING_ENCOUNTER'
    END AS encounter_type
FROM service_detail sd
JOIN cpt c ON c.cpt_id = sd.service_detail_cptid
WHERE sd.service_detail_patientid = 17568
  AND sd.service_detail_dos BETWEEN '2025-01-01' AND '2025-12-31'
ORDER BY visit_date, encounter_type;
```

---

# 🧬 3. Clinical Elements Extraction

## 🎯 Purpose

To retrieve all clinical data recorded during encounter.

---

## 🧾 SQL Query

```sql
SELECT patient_clinical_elements_gwid,
       patient_clinical_elements_value
FROM patient_clinical_elements
WHERE patient_clinical_elements_patientid = 17568
  AND patient_clinical_elements_encounterid = 4209
  AND patient_clinical_elements_chartid = 17475;
```

---

# 🔗 4. Clinical Element → SNOMED Mapping

## 🎯 Purpose

To verify mapping of patient result values to SNOMED codes.

---

## 🧾 SQL Query

```sql
SELECT
  pce.patient_clinical_elements_gwid,
  pce.patient_clinical_elements_value,
  ceo.clinical_elements_options_name,
  ceo.clinical_elements_options_snomed
FROM patient_clinical_elements pce
LEFT JOIN clinical_elements_options ceo
  ON ceo.clinical_elements_options_gwid = pce.patient_clinical_elements_gwid
 AND ceo.clinical_elements_options_value = pce.patient_clinical_elements_value
WHERE pce.patient_clinical_elements_patientid = 17568
  AND pce.patient_clinical_elements_encounterid = 4209
  AND pce.patient_clinical_elements_chartid = 17475;
```

---

# 🔎 5. LOINC Code Validation (Example: Depression Screening)

## 🎯 Purpose

To confirm correct mapping to LOINC codes.

LOINC OID:

```
2.16.840.1.113883.6.1
```

---

## 🧾 SQL Query

```sql
SELECT 
    p.patient_clinical_elements_patientid AS patient_id,
    p.patient_clinical_elements_gwid AS gwid,
    p.patient_clinical_elements_value AS patient_result,
    c.cnm_code_system_code AS code,
    c.cnm_code_system_oid AS code_system_oid
FROM patient_clinical_elements p
JOIN cnm_code_system c
  ON p.patient_clinical_elements_gwid = c.cnm_code_system_gwid
WHERE c.cnm_code_system_oid = '2.16.840.1.113883.6.1'
  AND c.cnm_code_system_code = '72166-2'
  AND p.patient_clinical_elements_patientid = 17568;
```

---

# 🩺 6. Care Plan Intervention Validation

## 🎯 Purpose

To check interventions recorded during encounter.

---

## 🧾 SQL Query

```sql
SELECT
  careplan_intervention_id,
  careplan_intervention_description,
  careplan_intervention_code,
  careplan_intervention_code_system,
  careplan_intervention_codesetoid,
  careplan_intervention_performed_on
FROM careplan_intervention
WHERE careplan_intervention_patient_id = 17568
  AND careplan_intervention_encounter_id = 4209;
```

---

# 📅 7. Encounter Status Validation

## 🎯 Purpose

To ensure only **active encounters** are considered.

---

## 🧾 SQL Query

```sql
SELECT
    pce.patient_clinical_elements_patientid AS patient_id,
    pce.patient_clinical_elements_encounterid AS encounter_id,
    e.encounter_date,
    e.encounter_status
FROM patient_clinical_elements pce
JOIN encounter e
  ON e.encounter_id = pce.patient_clinical_elements_encounterid
WHERE pce.patient_clinical_elements_patientid = 17568
  AND e.encounter_status <> 100;
```

---

# 📊 8. QDM Data Extraction Query

## 🎯 Purpose

To extract final QDM-ready clinical data for eCQM calculation.

---

## 🧾 SQL Query

```sql
SELECT
    pce.patient_clinical_elements_id AS qdm_id,
    pce.patient_clinical_elements_patientid AS patient_id,
    pce.patient_clinical_elements_encounterid AS encounter_id,
    pce.patient_clinical_elements_gwid AS gwid,
    ccs.cnm_code_system_code AS loinc_code,
    pce.patient_clinical_elements_value AS patient_result,
    ceo.clinical_elements_options_snomed AS result_code,
    e.encounter_date AS recorded_date
FROM patient_clinical_elements pce
JOIN clinical_elements ce
  ON ce.clinical_elements_gwid = pce.patient_clinical_elements_gwid
 AND ce.clinical_elements_isactive = true
JOIN cnm_code_system ccs
  ON ccs.cnm_code_system_gwid = ce.clinical_elements_gwid
LEFT JOIN clinical_elements_options ceo
  ON ceo.clinical_elements_options_gwid = pce.patient_clinical_elements_gwid
JOIN encounter e
  ON e.encounter_id = pce.patient_clinical_elements_encounterid
WHERE pce.patient_clinical_elements_patientid = 17568;
```

---
