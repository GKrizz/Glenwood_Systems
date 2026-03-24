
# 📘 README: Fixing Statin Allergy Misclassification (eCQM CMS347v8)

---

## 🎯 Objective

Ensure that **patients with statin allergy** are correctly classified as:

> ✅ **Denominator Exception**
> ❌ NOT incorrectly shown as **NOT MET**

---

## 🧠 Background

As per **CMS347v8 (Statin Therapy Measure)**:

### ✔ Denominator Exceptions include:

* Statin Allergy
* Statin-associated muscle symptoms
* Hospice / Palliative care
* ESRD
* Liver disease

👉 Specifically:

```cql
exists ["Allergy/Intolerance": "Statin Allergen"]
```

---

## 📦 Value Set Details

* **Value Set Name:** Statin Allergen
* **OID:** `2.16.840.1.113762.1.4.1110.42`
* **Code Systems:** RXNORM, SNOMEDCT

### ✅ Valid Codes

```sql
301542  -- rosuvastatin
36567   -- simvastatin
41127   -- fluvastatin
42463   -- pravastatin
6472    -- lovastatin
83367   -- atorvastatin
861634  -- pitavastatin
372912004 -- SNOMED
96302009  -- SNOMED
```

---

## 🚨 Problem Summary

Patients **with statin allergy are appearing as NOT MET**.

### 🔍 Root Cause

From analysis:

| Issue Type     | Example Value      | Impact         |
| -------------- | ------------------ | -------------- |
| NULL Code      | `NULL`             | ❌ Not detected |
| Undefined Code | `'undefined'`      | ❌ Not detected |
| Free-text only | `'STATINS'`        | ❌ Not detected |
| Wrong Code     | Non-valueset codes | ❌ Not detected |

👉 eCQM engine only evaluates coded entries:

```sql
WHERE patallerg_allergy_code IN (value set)
```

---

## 📊 Data Model Context



```
lab_description (master)
    ↓
lab_entries (patient test instance)
    ↓
lab_entries_parameter (actual values)
    ↓
lab_parameters (parameter master)
```

👉 Similarly:

```
patient_registration → chart → patient_allergies
```

---

## 🔬 Example Problem Case

**Account:** 026989
**Chart ID:** 21666

```sql
SELECT patallerg_allergicto, patallerg_allergy_code
FROM patient_allergies
WHERE patallerg_chartid = 21666;
```

### Result:

```text
PRAVASTATIN   → undefined
ROSUVASTATIN  → undefined
```

❌ Not counted → becomes **NOT MET**

---

## ✅ Expected Behavior

After correct coding:

```text
PRAVASTATIN  → 42463
ROSUVASTATIN → 301542
```

✔ Patient becomes **Denominator Exception**

---

# 🛠️ Solution

---

## 1️⃣ Identify Problem Records

```sql
SELECT *
FROM patient_allergies
WHERE patallerg_status = 1
AND (
    patallerg_allergicto ILIKE '%statin%'
    OR patallerg_allergicto ILIKE '%vastatin%'
)
AND (
    patallerg_allergy_code IS NULL
    OR patallerg_allergy_code = 'undefined'
    OR patallerg_allergy_code NOT IN (
        '301542','36567','41127','42463',
        '6472','83367','861634','372912004','96302009'
    )
);
```

---

## 2️⃣ Fix Missing / Undefined Codes

```sql
UPDATE patient_allergies
SET patallerg_allergy_code =
    CASE
        WHEN patallerg_allergicto ILIKE '%rosuvastatin%' THEN '301542'
        WHEN patallerg_allergicto ILIKE '%pravastatin%' THEN '42463'
        WHEN patallerg_allergicto ILIKE '%atorvastatin%' THEN '83367'
        WHEN patallerg_allergicto ILIKE '%simvastatin%' THEN '36567'
        WHEN patallerg_allergicto ILIKE '%fluvastatin%' THEN '41127'
        WHEN patallerg_allergicto ILIKE '%lovastatin%' THEN '6472'
        WHEN patallerg_allergicto ILIKE '%pitavastatin%' THEN '861634'
    END,
    patallerg_codesystem = '2.16.840.1.113883.6.88'
WHERE patallerg_status = 1
AND (
    patallerg_allergy_code IS NULL
    OR patallerg_allergy_code = 'undefined'
);
```

---

## 3️⃣ Handle Generic "STATINS"

```sql
UPDATE patient_allergies
SET patallerg_allergy_code = '83367', -- fallback
    patallerg_codesystem = '2.16.840.1.113883.6.88'
WHERE patallerg_allergicto ILIKE '%statin%'
AND (
    patallerg_allergy_code IS NULL
    OR patallerg_allergy_code = 'undefined'
);
```

---

## 4️⃣ Validate Fix

```sql
SELECT
    patallerg_chartid,
    STRING_AGG(patallerg_allergicto, ', ') AS allergens,
    STRING_AGG(patallerg_allergy_code, ', ') AS codes
FROM patient_allergies
WHERE patallerg_allergy_code IN (
    '301542','36567','41127','42463',
    '6472','83367','861634','372912004','96302009'
)
GROUP BY patallerg_chartid;
```

---

## 5️⃣ Measure Logic Check (Simplified)

```sql
SELECT DISTINCT c.chart_id
FROM chart c
JOIN patient_allergies pa
    ON pa.patallerg_chartid = c.chart_id
WHERE pa.patallerg_status = 1
AND pa.patallerg_allergy_code IN (
    '301542','36567','41127','42463',
    '6472','83367','861634','372912004','96302009'
);
```

👉 These patients = **Denominator Exception**

---

# 🔒 Prevention (Future Data Quality)

## ✅ Enforce at Application Level

* Allergy entry MUST include:

  * Code
  * Code system
* Disallow:

  * NULL codes
  * `'undefined'`
  * free-text without mapping

---

## ✅ Add DB Constraint (Optional)

```sql
ALTER TABLE patient_allergies
ADD CONSTRAINT chk_valid_statin_code
CHECK (
    patallerg_allergy_code IS NOT NULL
);
```

---

## ✅ Monitoring Query

```sql
SELECT COUNT(*)
FROM patient_allergies
WHERE patallerg_allergicto ILIKE '%statin%'
AND (
    patallerg_allergy_code IS NULL
    OR patallerg_allergy_code = 'undefined'
);
```

---

# 📌 Key Takeaways

* eCQM logic is **code-driven**, not text-driven
* Missing codes = allergy **does not exist** for measure
* This is a **data issue**, not a measure logic issue
* Fixing codes → immediately fixes NOT MET problem

---

# 🏁 Final Outcome

After implementing fixes:

| Before    | After                   |
| --------- | ----------------------- |
| NOT MET ❌ | Denominator Exception ✅ |

---
