# 📌 Case #240728 — URGENT: MIPS (CMS22v13 – Measure 317)

**Account:** California Huntington Physicians
**Product:** GlacePremium
**Module:** MIPS
**Measure:** CMS22v13 – 317
**Title:** Preventive Care and Screening: Screening for High Blood Pressure and Follow-Up Documented
**Priority:** 🔥 High
**ETA:** 2 hours

**Case Owner:** Idhayavani B
**Assigned To:** Gobala Krishnan

---

## ✅ Status Summary

* ✔ Not Met count issue fixed
* ✔ Development commit completed
* ✔ Testing team notified
* ✔ Loom video shared
* ⏳ Move fix to Production

---

## 🧩 Problem Statement

In **MIPS Measure 317**, the **Pencil (Document) icon** behavior and **Not Met → Met transition** were inconsistent in multi-encounter scenarios.
Specifically:

* Numerator was not updating correctly after documenting required follow-up.
* Pencil icon was not consistently enabled/disabled based on compliance.
* BP elements mapping had missing entries in `cnm_code_system` for Sitting BP.

---

## 🔗 Environment & Test Links

**Development Environment**

```
https://dev2.glaceemr.com:444/D2Desktop/jsp/chart/patientdetails/Patient_Chart.Action
```

**Test Patient Details**

* `patientId = 1492`
* `chartId = 1497`
* `providerId = 1286`
* Reporting Year: **2026**

---

## 🧠 Measure Configuration Validation

### Provider Measure Mapping

```sql
SELECT
  ep.emp_profile_fullname AS provider_name,
  mpc.macra_provider_configuration_reporting_year AS year,
  qmpm.quality_measures_provider_mapping_measure_id AS measure_id,
  md.title AS measure_name
FROM macra_provider_configuration mpc
JOIN quality_measures_provider_mapping qmpm
  ON qmpm.quality_measures_provider_mapping_provider_id = mpc.macra_provider_configuration_provider_id
 AND qmpm.quality_measures_provider_mapping_reporting_year = mpc.macra_provider_configuration_reporting_year
LEFT JOIN measure_details md
  ON md.measure_id = qmpm.quality_measures_provider_mapping_measure_id
LEFT JOIN emp_profile ep
  ON ep.emp_profile_empid = mpc.macra_provider_configuration_provider_id
WHERE mpc.macra_provider_configuration_provider_id = 1286
  AND mpc.macra_provider_configuration_reporting_year = 2026;
```

✔ **Measure 317 correctly mapped to provider for 2026**

---

## 👤 Patient Eligibility Validation

```sql
SELECT
  pr.patient_registration_id,
  pr.patient_registration_dob,
  EXTRACT(YEAR FROM AGE('2026-01-01', pr.patient_registration_dob)) AS age,
  CASE
    WHEN EXTRACT(YEAR FROM AGE('2026-01-01', pr.patient_registration_dob)) BETWEEN 18 AND 85
    THEN 'ELIGIBLE'
    ELSE 'NOT ELIGIBLE'
  END AS age_eligibility
FROM patient_registration pr
WHERE pr.patient_registration_id = 1492;
```

✔ **Patient Age = 18 → Eligible**

---

## 🏥 Encounter & CPT Qualification

### Encounters

```sql
SELECT
  e.encounter_id,
  e.encounter_date,
  e.encounter_service_doctor AS provider_id,
  ep.emp_profile_fullname
FROM encounter e
JOIN chart c
  ON c.chart_id = e.encounter_chartid
LEFT JOIN emp_profile ep
  ON ep.emp_profile_empid = e.encounter_service_doctor
WHERE c.chart_patientid = 1492
ORDER BY e.encounter_date;
```

### Qualifying CPT Codes

```sql
SELECT
  sd.service_detail_id,
  sd.service_detail_patientid AS patient_id,
  sd.service_detail_dos AS dos,
  sd.service_detail_sdoctorid AS provider_id,
  c.cpt_cptcode
FROM service_detail sd
JOIN cpt c
  ON c.cpt_id = sd.service_detail_cptid
WHERE sd.service_detail_patientid = 1492
  AND sd.service_detail_dos BETWEEN '2026-01-01' AND '2026-12-31'
  AND c.cpt_cptcode IN ('99202','99203','99204','99205','99212','99213','99214','99215');
```

✔ **Encounters 2808 & 3270 qualify**

---

## 🩺 Vitals (BP) Validation

### LOINC Mapping

| BP Type      | LOINC  |
| ------------ | ------ |
| Systolic BP  | 8480-6 |
| Diastolic BP | 8462-4 |

### BP Extraction per Encounter

```sql
SELECT
  pce.patient_clinical_elements_encounterid,
  MAX(CASE WHEN cs.cnm_code_system_code = '8480-6'
           THEN pce.patient_clinical_elements_value END) AS systolic_bp,
  MAX(CASE WHEN cs.cnm_code_system_code = '8462-4'
           THEN pce.patient_clinical_elements_value END) AS diastolic_bp
FROM patient_clinical_elements pce
JOIN cnm_code_system cs
  ON cs.cnm_code_system_gwid = pce.patient_clinical_elements_gwid
WHERE pce.patient_clinical_elements_patientid = 1492
GROUP BY pce.patient_clinical_elements_encounterid;
```

| Encounter | BP       |
| --------- | -------- |
| 2808      | 118 / 78 |
| 3270      | 130 / 90 |

---

## 🛠️ Fix Applied

### Missing LOINC Mapping (Sitting BP)

```sql
INSERT INTO cnm_code_system (
  cnm_code_system_gwid,
  cnm_code_system_code,
  cnm_code_system_oid,
  cnm_code_system_isactive
) VALUES (
  '0000200200100026000',
  '8480-6',
  '2.16.840.1.113883.6.1',
  true
);
```

✔ Ensured BP values are correctly evaluated by MIPS engine.

---

## ✏️ Pencil Icon & Numerator Validation

### Measure Entry Validation

```sql
SELECT
  quality_measures_patient_entries_ipp,
  quality_measures_patient_entries_denominator,
  quality_measures_patient_entries_denominator_exclusion,
  quality_measures_patient_entries_numerator,
  quality_measures_patient_entries_updated_on
FROM quality_measures_patient_entries
WHERE quality_measures_patient_entries_patient_id = 1492
  AND quality_measures_patient_entries_provider_id = 1286
  AND quality_measures_patient_entries_reporting_year = 2026
  AND quality_measures_patient_entries_measure_id = '317';
```

### Final Measure Outcome

| Encounter | BP       | Required Action          | Status |
| --------- | -------- | ------------------------ | ------ |
| 2808      | 118 / 78 | Counseling + Follow-up   | ✅ MET  |
| 3270      | 130 / 90 | Lifestyle + FU ≤ 4 weeks | ✅ MET  |

✔ Numerator incremented only after documentation
✔ Pencil icon enabled for unmet encounter
✔ Pencil icon disabled post-compliance

---

## 🎯 Final Result

* **IPP & Denominator increment per qualifying encounter**
* **Numerator updates only on documented intervention**
* **Multi-encounter logic validated**
* **UI (Pencil icon) behavior aligned with compliance**
* **Fully compliant with CMS22v13 specification**

---
