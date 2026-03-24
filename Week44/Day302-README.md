
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
