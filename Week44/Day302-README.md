
# 📘 README: MSE Tab Not Loading (BPC – Case #243359)

---

# 🎯 Objective

Fix **MSE (Measure Summary & Entries) tab not loading** for:

| Field    | Value |
| -------- | ----- |
| Provider | 25    |
| Measure  | 226   |
| Year     | 2025  |

---

# 🚨 Issue Summary

* MIPS Flowsheet loads ✅
* But **MSE tab does NOT load ❌**
* No visible UI error (silent failure)

---

# 🧩 System Flow

```text
MSE Tab Click
     ↓
Fetch Patient Entries
     ↓
quality_measures_patient_entries
     ↓
Aggregate + Rate Calculation
     ↓
macra_measures_rate
     ↓
Render UI
```

---

# 🔍 Debug Checklist

You already verified:

---

## ✅ 1. Provider Mapping Exists

```sql
SELECT * 
FROM quality_measures_provider_mapping 
WHERE provider_id = 25 
AND measure_id = 226 
AND year = 2025;
```

✔ Required for measure visibility

---

## ✅ 2. Provider Config Exists

```sql
SELECT * 
FROM macra_provider_configuration 
WHERE provider_id = 25 
AND reporting_year = 2025;
```

✔ Required for flowsheet

---

## ⚠️ 3. Patient Entries (CRITICAL)

```sql
SELECT * 
FROM quality_measures_patient_entries 
WHERE provider_id = 25 
AND measure_id = 226 
AND reporting_year = 2025;
```

👉 This table drives **MSE Tab**

---

## ⚠️ 4. Aggregated Data

```sql
SELECT 
    SUM(ipp),
    SUM(denominator),
    SUM(numerator),
    SUM(denominator_exception)
FROM quality_measures_patient_entries
WHERE provider_id = 25
AND measure_id = 226
AND reporting_year = 2025;
```

---

## ❗ 5. macra_measures_rate (FINAL LAYER)

```sql
SELECT * 
FROM macra_measures_rate
WHERE provider_id = 25 
AND measure_id = 226 
AND reporting_year = 2025;
```

👉 If missing → MSE tab fails

---

# 🧠 Root Cause Possibilities

---

## 🔴 Case 1: No Data in `macra_measures_rate`

👉 Most common cause

* MSE tab expects aggregated record
* If missing → UI breaks

---

## 🔴 Case 2: Empty Patient Entries

* No rows → no aggregation
* UI cannot render grid

---

## 🔴 Case 3: Data Mismatch

| Table               | Issue       |
| ------------------- | ----------- |
| patient_entries     | Data exists |
| macra_measures_rate | Missing     |

👉 Sync issue

---

## 🔴 Case 4: Criteria Mismatch

```sql
criteria != expected
```

👉 UI filters out rows

---

# 🛠️ FIX PLAN

---

# ✅ FIX 1: Ensure Patient Entries Exist

```sql
SELECT COUNT(*) 
FROM quality_measures_patient_entries
WHERE provider_id = 25
AND measure_id = 226
AND reporting_year = 2025;
```

👉 If **0 → regenerate measure**

---

# ✅ FIX 2: Populate `macra_measures_rate`

---

## 🔧 Insert (Manual Fix)

```sql
INSERT INTO macra_measures_rate (
    provider_id,
    measure_id,
    reporting_year,
    ipp,
    denominator,
    numerator,
    denominator_exception,
    performance_rate
)
SELECT
    provider_id,
    measure_id,
    reporting_year,
    SUM(ipp),
    SUM(denominator),
    SUM(numerator),
    SUM(denominator_exception),
    ROUND(
        SUM(numerator) * 100.0 / NULLIF(SUM(denominator), 0),
        2
    )
FROM quality_measures_patient_entries
WHERE provider_id = 25
AND measure_id = 226
AND reporting_year = 2025
GROUP BY provider_id, measure_id, reporting_year;
```

---

# ✅ FIX 3: Recalculate Measure (Preferred)

Instead of manual insert:

```text
Run MIPS Calculation Job
```

OR

```text
Trigger:
MIPSPerformanceReport.Action?mode=1
```

---

# ✅ FIX 4: Validate Data Consistency

---

## 🔎 Cross Check

```sql
-- Entries
SELECT COUNT(*) FROM quality_measures_patient_entries
WHERE provider_id=25 AND measure_id=226 AND reporting_year=2025;

-- Rate
SELECT COUNT(*) FROM macra_measures_rate
WHERE provider_id=25 AND measure_id=226 AND reporting_year=2025;
```

✔ Both must exist

---

# 🧪 Validation

---

## ✅ Expected Data

| Field | Value |
| ----- | ----- |
| IPP   | > 0   |
| DENOM | > 0   |
| NUMER | ≥ 0   |

---

## ✅ UI Behavior

| Before          | After                  |
| --------------- | ---------------------- |
| MSE tab blank ❌ | MSE loads ✅            |
| No data ❌       | Patient list visible ✅ |

---

# 🛡️ Preventive Fix

---

## 1️⃣ Always Sync Tables

```text
patient_entries → macra_measures_rate
```

---

## 2️⃣ Add Fallback in Code

```java
if(rate == null){
   calculateOnTheFly();
}
```

---

## 3️⃣ Add Logs

```java
log.info("MSE load → entries count: " + count);
```

---

# 🏁 Final Summary

| Issue               | Root Cause              | Fix                          |
| ------------------- | ----------------------- | ---------------------------- |
| MSE tab not loading | Missing aggregated data | Populate macra_measures_rate |
| UI blank            | No patient entries      | Regenerate data              |

---





# 📘 README: MIPS Report Update – MDD + Suicide Risk Assessment (Case: JRANA)

---

# 🎯 Objective

Update **MIPS Measure (MDD – Suicide Risk Assessment)** for **Reporting Year 2025**

| Field      | Value                       |
| ---------- | --------------------------- |
| Patient ID | 11396                       |
| Chart ID   | 11600                       |
| Measure    | MDD Suicide Risk Assessment |
| Age        | 15 ✅                        |
| Status     | ❌ NOT MET                   |

---

# 📌 Issue Summary

Even though:

* ✅ Patient has **MDD diagnosis (F33.2)**
* ✅ Patient has **valid encounters (CPT: 90791, 99214)**
* ✅ Suicide Risk Assessment is documented

👉 Still showing **NOT MET**

---

# 🔍 Root Cause

👉 **Timing Issue**

```text
Suicide Risk Assessment created on:
2026-03-11 ❌ (Outside Measurement Period)

Measurement Period:
2025-01-01 → 2025-12-31
```

So system evaluates:

| Condition   | Result |
| ----------- | ------ |
| IPP         | ✅      |
| Denominator | ✅      |
| Numerator   | ❌      |

---

# 🧠 Measure Logic (Simplified)

---

## ✅ Initial Population (IPP)

```sql
Age between 6–16
```

✔ Patient age = 15 → INCLUDED

---

## ✅ Denominator

```sql
Valid Encounter + MDD Diagnosis
```

✔ CPT codes present
✔ ICD10 F33.2 present

---

## ❌ Numerator

```sql
Suicide Risk Assessment (SNOMED: 225337009)
WITHIN encounter period
AND within measurement year
```

❌ Recorded in **2026 → NOT counted**

---

# 📊 Evidence

---

## 🔹 Encounters

```sql
SELECT service_detail_dos, cpt_cptcode
FROM service_detail sd
JOIN cpt c ON c.cpt_id = sd.service_detail_cptid
WHERE sd.service_detail_patientid = 11396;
```

✔ Valid encounters found

---

## 🔹 Diagnosis

```sql
SELECT patient_assessments_dxcode
FROM patient_assessments
WHERE patient_assessments_patientid = 11396;
```

✔ F33.2 present

---

## 🔹 Suicide Risk Assessment

```sql
SELECT patient_clinical_elements_created_on
FROM patient_clinical_elements
WHERE patient_id = 11396
AND SNOMED = '225337009';
```

❌ Result:

```
2026-03-11 ❌
```

---

# 🛠️ FIX OPTIONS

---

# ✅ OPTION 1: Correct Data (RECOMMENDED)

👉 If documentation was actually done in 2025

### 🔧 Update Date

```sql
UPDATE patient_clinical_elements
SET patient_clinical_elements_created_on = '2025-12-15'
WHERE patient_clinical_elements_id = 10094868;
```

---

# ⚠️ OPTION 2: Re-document in Correct Encounter

👉 Preferred clinical approach

* Open encounter (2025)
* Add **Suicide Risk Assessment**
* Save again

---

# ❌ OPTION 3: Do Nothing

👉 Measure remains:

```text
DENOMINATOR = YES
NUMERATOR   = NO
STATUS      = NOT MET ❌
```

---

# 🔄 Recalculate MIPS

After fix:

```text
Trigger:
MIPSPerformanceReport.Action?mode=1
```

OR

```text
Run MIPS Batch Job
```

---

# 🧪 Validation Query

---

## ✅ Final Check

```sql
SELECT
pce.patient_clinical_elements_patientid,
pce.patient_clinical_elements_created_on,
ce.clinical_elements_snomed
FROM patient_clinical_elements pce
JOIN clinical_elements ce
ON ce.clinical_elements_gwid = pce.patient_clinical_elements_gwid
WHERE ce.clinical_elements_snomed = '225337009'
AND pce.patient_clinical_elements_patientid = 11396
AND pce.patient_clinical_elements_created_on
BETWEEN '2025-01-01' AND '2025-12-31';
```

✔ Should return **1 row**

---

# 🛡️ Preventive Fix (Important)

---

## 🔴 Problem Pattern

```text
Clinical data entered AFTER year-end
→ Measure fails
```

---

## ✅ Solution

### 1️⃣ Add Validation in UI

```text
Block future-date documentation for past encounters
```

---

### 2️⃣ Add Backend Check

```java
if(date not in measurementPeriod){
   excludeFromNumerator();
}
```

---

#### 3️⃣ Logging

```java
log.warn("Assessment outside measurement period");
```

---

# 🏁 Final Summary

| Layer      | Status         |
| ---------- | -------------- |
| Age        | ✅              |
| Encounter  | ✅              |
| Diagnosis  | ✅              |
| Assessment | ❌ (Wrong year) |

---
