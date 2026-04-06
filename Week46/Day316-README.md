
# Smoking Cessation Intervention 

## 📌 Purpose

This script identifies **patients with smoking status (current smokers)** in 2025 and ensures that **Smoking Cessation Education interventions** are properly recorded in the `careplan_intervention` table.

It also avoids duplicate entries and supports rollback if needed.

---

## 🧠 Business Logic

### ✅ Include Patients:

* Patients with **Smoking Status (GWID: `0000100303000000013`)**
* Encounter date within **2025**
* Status indicates **current smoker**, such as:

  * Cigarette smoker
  * Light / Moderate / Heavy smoker
  * Occasional smoker
  * Smoker

### ❌ Exclude:

* Ex-smokers
* Non-smokers
* Quit / Never / Denies smoking

---

## 🔍 Step 1: Identify Eligible Patients

```sql
SELECT DISTINCT ON (pce.patient_clinical_elements_patientid)
    pce.patient_clinical_elements_patientid AS patient_id,
    e.encounter_id,
    e.encounter_date
FROM patient_clinical_elements pce
JOIN encounter e 
    ON e.encounter_id = pce.patient_clinical_elements_encounterid
JOIN clinical_elements_options ceo 
    ON ceo.clinical_elements_options_gwid = pce.patient_clinical_elements_gwid
   AND ceo.clinical_elements_options_value = pce.patient_clinical_elements_value
WHERE pce.patient_clinical_elements_gwid = '0000100303000000013'
  AND e.encounter_date BETWEEN '2025-01-01' AND '2025-12-31'
  AND ceo.clinical_elements_options_name ILIKE '%smoker%'
  AND ceo.clinical_elements_options_name NOT ILIKE '%ex-%'
  AND ceo.clinical_elements_options_name NOT ILIKE '%non%'
  AND ceo.clinical_elements_options_name NOT ILIKE '%quit%'
  AND ceo.clinical_elements_options_name NOT ILIKE '%never%'
  AND ceo.clinical_elements_options_name NOT ILIKE '%denies%'
ORDER BY pce.patient_clinical_elements_patientid, e.encounter_date DESC;
```

---

## 🛠️ Step 2: Insert Missing Interventions

### Inserts 2 interventions:

* Smoking cessation education
* Tobacco use cessation education (procedure)

```sql
INSERT INTO careplan_intervention (
    careplan_intervention_patient_id,
    careplan_intervention_encounter_id,
    careplan_intervention_description,
    careplan_intervention_code,
    careplan_intervention_code_system,
    careplan_intervention_code_system_name,
    careplan_intervention_status,
    careplan_intervention_performed_by,
    careplan_intervention_performed_on,
    careplan_intervention_created_by,
    careplan_intervention_created_on
)
SELECT 
    s.patient_id,
    s.encounter_id,
    i.description,
    i.code,
    i.code_system,
    i.code_system_name,
    2,              -- performed
    19,             -- provider_id
    s.encounter_date,
    19,
    NOW()
FROM (
    -- Latest smoking encounter per patient
    SELECT DISTINCT ON (pce.patient_clinical_elements_patientid)
        pce.patient_clinical_elements_patientid AS patient_id,
        e.encounter_id,
        e.encounter_date
    FROM patient_clinical_elements pce
    JOIN encounter e 
        ON e.encounter_id = pce.patient_clinical_elements_encounterid
    JOIN clinical_elements_options ceo 
        ON ceo.clinical_elements_options_gwid = pce.patient_clinical_elements_gwid
       AND ceo.clinical_elements_options_value = pce.patient_clinical_elements_value
    WHERE pce.patient_clinical_elements_gwid = '0000100303000000013'
      AND e.encounter_date BETWEEN '2025-01-01' AND '2025-12-31'
      AND ceo.clinical_elements_options_name ILIKE '%smoker%'
      AND ceo.clinical_elements_options_name NOT ILIKE '%ex-%'
      AND ceo.clinical_elements_options_name NOT ILIKE '%non%'
      AND ceo.clinical_elements_options_name NOT ILIKE '%quit%'
      AND ceo.clinical_elements_options_name NOT ILIKE '%never%'
      AND ceo.clinical_elements_options_name NOT ILIKE '%denies%'
    ORDER BY pce.patient_clinical_elements_patientid, e.encounter_date DESC
) s
CROSS JOIN (
    VALUES 
        ('Smoking cessation education', '225323000', '2.16.840.1.113883.6.96', 'SNOMED'),
        ('Tobacco use cessation education (procedure)', '702388001', '2.16.840.1.113883.6.96', 'SNOMEDCT')
) i (description, code, code_system, code_system_name)
WHERE NOT EXISTS (
    SELECT 1 
    FROM careplan_intervention ci
    WHERE ci.careplan_intervention_patient_id = s.patient_id
      AND ci.careplan_intervention_encounter_id = s.encounter_id
      AND ci.careplan_intervention_description = i.description
);
```

---

## 🔒 Step 3: Backup Before Changes

```sql
\copy (
  SELECT * 
  FROM careplan_intervention
  WHERE careplan_intervention_patient_id IN (<PATIENT_IDS>)
) TO '/tmp/backup_careplan_fix.csv' WITH CSV HEADER;
```

---

## ❌ Step 4: Rollback (if needed)

```sql
DELETE FROM careplan_intervention
WHERE careplan_intervention_patient_id IN (<PATIENT_IDS>)
AND careplan_intervention_created_by = 19
AND careplan_intervention_created_on >= NOW() - INTERVAL '10 minutes';
```

---

## ✅ Step 5: Validation

### Count inserted records

```sql
SELECT COUNT(*)
FROM careplan_intervention
WHERE careplan_intervention_patient_id IN (<PATIENT_IDS>);
```

### Verify inserted data

```sql
SELECT 
    careplan_intervention_patient_id,
    careplan_intervention_encounter_id,
    careplan_intervention_description,
    careplan_intervention_performed_on
FROM careplan_intervention
WHERE careplan_intervention_patient_id IN (<PATIENT_IDS>)
ORDER BY careplan_intervention_patient_id;
```

---

## 📊 Expected Outcome

* Each patient should have **2 records per encounter**:

  * Smoking cessation education
  * Tobacco use cessation education
* Example:

```
patient_id | encounter_id | description
-----------------------------------------
4704       | 122683       | Smoking cessation education
4704       | 122683       | Tobacco use cessation education
```

---

## ⚠️ Notes

* `provider_id = 19` is used for insertion
* `status = 2` → indicates **performed**
* Script is **idempotent** (won’t insert duplicates)
* Works only for **2025 reporting year**

---
