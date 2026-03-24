
#  MIPS Flowsheet Issues (Smoking Status Not Saving & MACRA Config Missing)

---

# 🧾 Document Scope

This README covers **Day 299 issues**:

1️⃣ **WHRA Case (#243305)** → Smoking Status not reflecting in MIPS
2️⃣ **NEPA Case (#243281)** → MIPS Flowsheet not loading (MACRA config issue) 

---

# 🧩 ISSUE 1: Smoking Status Not Saving (WHRA)

## 🔴 Problem Statement

* Smoking status updated in UI ✅
* Saved successfully message shown ✅
* But MIPS shows **“Not Met” ❌**

---

## 👤 Patient Context

| Field      | Value                             |
| ---------- | --------------------------------- |
| Patient ID | 7535                              |
| Age        | 52                                |
| Measure    | Tobacco Use Screening & Cessation |

---

## 🔍 Data Findings

### ✅ Smoking Status Exists in DB

```sql
GWID: 0000100303000000015
LOINC: 72166-2
Value: Chews tobacco
SNOMED: 81703003
```

✔ Stored correctly
✔ Mapped to LOINC
✔ Has SNOMED

---

### ⚠️ Problematic Observation

At same encounter:

| GWID                | Value | Meaning                |
| ------------------- | ----- | ---------------------- |
| 0000100303000000015 | 3     | Chews tobacco          |
| 0000100303000000013 | 42    | Smoking status unknown |

👉 **Conflicting smoking statuses in same encounter**

---

## 🚨 Root Cause

### ❗ Issue 1: Multiple GWIDs for same concept

| GWID                  | Meaning                  |
| --------------------- | ------------------------ |
| `0000100303000000015` | Tobacco (Social History) |
| `0000100303000000013` | Smoking Status           |

👉 Measure logic likely expects **ONLY GWID = 13**

---

### ❗ Issue 2: Latest value selection issue

System may be picking:

```text
"Smoking status unknown" ❌
instead of
"Chews tobacco" ✅
```

---

### ❗ Issue 3: Measure logic mismatch

CMS logic expects:

* Valid SNOMED codes for smoking status
* Latest value within measurement period

But system:

* Either picks wrong GWID
* Or wrong record (ordering issue)

---

## 🛠️ FIX PLAN

---

### ✅ FIX 1: Restrict to Correct GWID

```sql
AND pce.patient_clinical_elements_gwid = '0000100303000000013'
```

---

### ✅ FIX 2: Always pick latest record

```sql
SELECT DISTINCT ON (patient_id)
...
ORDER BY patient_id, encounter_date DESC, last_modified DESC
```

---

### ✅ FIX 3: Valid Smoking SNOMED Filter

```sql
AND ceo.clinical_elements_options_snomed IN (
    '449868002', -- Current smoker
    '428041000124106',
    '8517006',
    '266919005',
    '77176002',
    '81703003',
    '8392000'
)
```

---

### ✅ FIX 4: Ignore “Unknown” if valid exists

```sql
CASE 
 WHEN EXISTS(valid_smoking) THEN ignore_unknown
END
```

---

## 🧪 Validation Query

```sql
SELECT DISTINCT ON (p.patient_registration_id)
    p.patient_registration_id,
    ceo.clinical_elements_options_name,
    ceo.clinical_elements_options_snomed,
    e.encounter_date
FROM patient_clinical_elements pce
JOIN clinical_elements_options ceo
  ON ceo.clinical_elements_options_gwid = pce.patient_clinical_elements_gwid
 AND ceo.clinical_elements_options_value = pce.patient_clinical_elements_value
JOIN encounter e
  ON e.encounter_id = pce.patient_clinical_elements_encounterid
JOIN patient_registration p
  ON p.patient_registration_id = pce.patient_clinical_elements_patientid
WHERE p.patient_registration_id = 7535
  AND pce.patient_clinical_elements_gwid = '0000100303000000013'
ORDER BY p.patient_registration_id, e.encounter_date DESC;
```

---

## ✅ Expected Outcome

| Before                | After                 |
| --------------------- | --------------------- |
| Not Met ❌             | Met ✅                 |
| Wrong status picked ❌ | Latest valid picked ✅ |

---

# 🧩 ISSUE 2: MACRA Not Configured (NEPA)

---

## 🔴 Error

```
MACRA isn't configured for this provider
```

---

## 👤 Context

| Field    | Value       |
| -------- | ----------- |
| Provider | 13 (CFritz) |
| Patient  | 135307      |
| Year     | 2025        |

---

## 🔍 Root Cause

### ❌ No data in:

```sql
quality_measures_provider_mapping
macra_provider_configuration
```

---

### 🔴 Critical Query

```sql
SELECT count(*)
FROM quality_measures_provider_mapping
WHERE provider_id = 13
AND reporting_year = 2025;
```

➡️ Result = **0**

👉 Hence:

```text
isMacraConfigured = false
```

---

## 🔁 Flow Breakdown

```
JSP Load
   ↓
Check Provider Mapping ❌
   ↓
Show Error Message
```

---

## 🛠️ FIX PLAN

---

### ✅ FIX 1: Insert Provider Mapping

```sql
INSERT INTO quality_measures_provider_mapping (
    provider_id,
    measure_id,
    reporting_year
)
SELECT
    13,
    measure_id,
    2025
FROM measure_details
WHERE is_active = true;
```

---

### ✅ FIX 2: Insert MACRA Config

```sql
INSERT INTO macra_provider_configuration (
    provider_id,
    reporting_year,
    reporting_method
)
VALUES (13, 2025, 2); -- ECQM
```

---

### ✅ FIX 3: Verify

```sql
SELECT count(*) 
FROM quality_measures_provider_mapping
WHERE provider_id = 13
AND reporting_year = 2025;
```

✔ Must be > 0

---

## 🧪 Validation

| Check                   | Expected |
| ----------------------- | -------- |
| Mapping exists          | ✅        |
| Reporting method exists | ✅        |
| Flowsheet loads         | ✅        |

---

# 🧠 Key Learnings

---

## 🔑 Smoking Status

* Multiple GWIDs → conflict
* Always pick **latest valid SNOMED**
* Ignore unknown if valid exists

---

## 🔑 MACRA Config

* Flowsheet depends on **provider mapping**
* No mapping → UI blocked
* Always verify config tables

---

# 🏁 Final Summary

| Issue                  | Root Cause                 | Fix                   |
| ---------------------- | -------------------------- | --------------------- |
| Smoking Status Not Met | Wrong GWID / record picked | Filter + latest logic |
| MACRA Not Configured   | Missing provider mapping   | Insert config data    |

---
