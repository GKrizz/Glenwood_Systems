
# 📘 CMS156v13 — Use of High-Risk Medications in Older Adults

## Measure ID: 238 — SQL Logic README

This document explains the **complete logic** for:

* ✅ Initial Population (IPP)
* ✅ Denominator Exclusions
* ✅ Numerator 1, 2, and 3 logic

for **CMS156v13**.

---

# 🧠 Measure Logic — Simple Explanation

This measure checks whether **patients age ≥ 65** are prescribed **high-risk medications**.

---

# 🟩 STEP 1 — Initial Population (IPP)

Patient must satisfy:

## ✅ Age ≥ 65 by end of Measurement Period

```sql id="age_check"
SELECT
  patient_registration_id,
  EXTRACT(YEAR FROM AGE('2025-12-31', patient_registration_dob)) AS age
FROM patient_registration
WHERE AGE('2025-12-31', patient_registration_dob) >= INTERVAL '65 years';
```

---

## ✅ At least one qualifying encounter in Measurement Period

```sql id="qualifying_cpt"
SELECT
    sd.service_detail_patientid,
    sd.service_detail_dos,
    c.cpt_cptcode
FROM service_detail sd
JOIN cpt c ON c.cpt_id = sd.service_detail_cptid
WHERE sd.service_detail_dos BETWEEN '2025-01-01' AND '2025-12-31'
AND c.cpt_cptcode IN (
'99202','99203','99204','99205','99212','99213','99214','99215',
'92002','92004','92012','92014',
'99395','99396','99397','99315','99316',
'99304','99305','99306','99307','99308','99309','99310',
'99324','99325','99326','99327','99328','99334','99335','99336','99337',
'99385','99386','99387',
'G0402','G0438','G0439',
'99341','99342','99344','99345','99347','99348','99349','99350',
'98966','98967','98968','99441','99442','99443',
'98970','98971','98972','98980','98981','99421','99422','99423',
'99457','99458','G0071','G2010','G2012','G2250','G2251','G2252',
'98969','99444','G2061','G2062','G2063'
);
```

---

# 🟥 STEP 2 — Denominator Exclusions

Exclude patients with:

---

## 1️⃣ Palliative Care Diagnosis

```sql id="palliative_dx"
SELECT *
FROM patient_assessments
WHERE patient_assessments_dxcode = 'Z51.5';
```

---

## 2️⃣ Hospice Encounters

```sql id="hospice_cpt"
SELECT *
FROM service_detail sd
JOIN cpt c ON c.cpt_id = sd.service_detail_cptid
WHERE c.cpt_cptcode IN (
'G9473','G9474','G9475','G9476','G9477','G9478','G9479',
'Q5003','Q5004','Q5005','Q5006','Q5007','Q5008','Q5010',
'S9126','T2042','T2043','T2044','T2045','T2046'
);
```

---

## 3️⃣ Hospice / Palliative SNOMED

```sql id="palliative_snomed"
SELECT *
FROM patient_assessments
WHERE patient_assessments_dxcode IN (
'385763009','385765002',
'170935008','170936009','305911006'
);
```

---

# 🟦 STEP 3 — Numerator Logic

The measure contains **3 numerator criteria**.

---

# 🟪 Numerator 1 — Multiple High-Risk Medications

## Rule:

Patient must have:

```text id="n1_rule"
≥ 2 prescriptions
FROM same high-risk medication class
ON different days
```

---

## SQL Logic

```sql id="numerator1"
SELECT
    doc_presc_patient_id,
    doc_presc_class_name,
    COUNT(DISTINCT DATE(doc_presc_start_date)) AS rx_days
FROM doc_presc
WHERE doc_presc_is_active = true
GROUP BY doc_presc_patient_id, doc_presc_class_name
HAVING COUNT(DISTINCT DATE(doc_presc_start_date)) >= 2;
```

---

# 🟨 Numerator 2 — Diagnosis Exception

Applies ONLY for:

* Antipsychotics
* Benzodiazepines

---

## Rule:

```text id="n2_rule"
If patient has valid diagnosis
within 1 year BEFORE prescription
→ REMOVE from numerator
```

---

## SQL Logic

```sql id="numerator2"
SELECT pa.patient_assessments_patientid
FROM patient_assessments pa
WHERE pa.patient_assessments_dxcode IN ('F20','F31','F41')
AND pa.patient_assessments_encounterdate >= CURRENT_DATE - INTERVAL '1 year';
```

---

# 🟩 Numerator 3 — FINAL RULE (Most Important)

This is the **actual measure numerator**.

---

## 🎯 Definition

Numerator3 = TRUE IF:

```text id="n3_logic"
(Numerator1 = TRUE AND Numerator2 = FALSE)
OR
Numerator2 = TRUE directly
```

---

## Meaning in Simple Words

Patient qualifies when:

✔ Has ≥ 2 high-risk medications
✔ Same drug class
✔ On different dates
✔ NO valid diagnosis to justify them

---

# 🧮 Final Numerator 3 SQL Logic

```sql id="numerator3"
WITH numerator1 AS (
    SELECT doc_presc_patient_id
    FROM doc_presc
    WHERE doc_presc_is_active = true
    GROUP BY doc_presc_patient_id, doc_presc_class_name
    HAVING COUNT(DISTINCT DATE(doc_presc_start_date)) >= 2
),

numerator2 AS (
    SELECT DISTINCT patient_assessments_patientid AS patient_id
    FROM patient_assessments
    WHERE patient_assessments_dxcode IN ('F20','F31','F41')
    AND patient_assessments_encounterdate >= CURRENT_DATE - INTERVAL '1 year'
)

SELECT
    n1.doc_presc_patient_id AS patient_id,
    CASE
        WHEN n1.doc_presc_patient_id NOT IN (SELECT patient_id FROM numerator2)
        THEN 'NUMERATOR 3 = TRUE'
        ELSE 'EXCLUDED BY DIAGNOSIS'
    END AS status
FROM numerator1 n1;
```

---

# 📊 Example Interpretation

For Patient **376094**:

```text id="example"
Age ≥ 65 → PASS
Qualifying CPT → PASS
Multiple high-risk meds → PASS
No exclusion diagnosis → PASS

FINAL RESULT = NUMERATOR 3 TRUE
```

---

# ⚠️ Common Implementation Mistakes

### ❌ Counting prescriptions on same date

Must count DISTINCT prescription dates.

---

### ❌ Ignoring drug class grouping

Measure checks same medication class.

---

### ❌ Missing 1-year diagnosis lookback

Diagnosis must be BEFORE medication.

---

# 🧩 Measure Flow

```text id="flow"
AGE ≥ 65
   ↓
Qualifying Visit
   ↓
High-Risk Medications ≥ 2
   ↓
Same Class?
   ↓
Diagnosis Present?
   ↓
NO → Numerator 3 TRUE
```
