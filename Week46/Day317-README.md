# Measure 134 (Depression Screening) NOT MET Issue

## 📌 Case Details

* **Case**: Day 317_Case#244024
* **Provider**: Dr. Lyo
* **Date**: March 31, 2026
* **Issue**: Measure 134 shows **NOT MET** even after screening is documented

---

## 🧾 Measure Info

* **Measure Name**: Preventive Care and Screening: Screening for Depression and Follow-Up Plan
* **Measure ID**: 134
* **Measurement Period**: 01-Jan-2025 → 31-Dec-2025

---

## 🧠 Measure Logic Summary

### ✅ Initial Population (IPP)

* Age ≥ 12 at start of measurement period
* At least one **qualifying encounter (CPT-based)**

✔ Patient **8601 → PASS**

---

### ❌ Denominator Exclusion

* Bipolar diagnosis before encounter

✔ No exclusion found

---

### 🎯 Numerator Logic

Patient must have:

#### Case 1: Negative Screening

* PHQ-9 result = **Negative**
* Within **14 days before encounter**

#### Case 2: Positive Screening

* PHQ-9 = Positive
* AND follow-up within:

  * Encounter date OR
  * Within **2 days after encounter**

---

## 🔍 Root Cause Analysis

### 🚨 Issue 1: Screening Date Outside Measurement Period

```sql
risk_assessment_performed_on = 2026-03-31
```

❌ Problem:

* Measure is calculated for **2025**
* Screening is recorded in **2026**

➡️ Result: **Ignored by measure logic**

---

### 🚨 Issue 2: Screening Not Within 14-Day Window

Expected:

```
Encounter Date: 2025-XX-XX  
Valid Screening Window: Encounter - 14 days → Encounter
```

Actual:

```
Screening Date: 2026-03-31 ❌
```

➡️ **Fails numerator condition**

---

### 🚨 Issue 3: Null performed_on Before Save

Before clicking save:

```sql
risk_assessment_performed_on = NULL
```

➡️ Measure engine ignores records with NULL date

---

### 🚨 Issue 4: CPT Auto Code Failure

API:

```
ScreeningAutoCpt → returns "-1"
```

➡️ CPT 96127 NOT generated
➡️ May affect encounter qualification / audit trail

---

## 🔬 Key SQL Checks

### 1️⃣ Verify Screening

```sql
SELECT *
FROM risk_assessment
WHERE risk_assessment_patient_id = 9514;
```

---

### 2️⃣ Check Qualifying Encounter

```sql
SELECT *
FROM service_detail sd
JOIN cpt c ON c.cpt_id = sd.service_detail_cptid
WHERE sd.service_detail_patientid = 9514
AND sd.service_detail_dos BETWEEN '2025-01-01' AND '2025-12-31';
```

---

### 3️⃣ Screening Window Validation

```sql
-- Must satisfy
screening_date BETWEEN encounter_date - 14 days AND encounter_date
```

---

### 4️⃣ Follow-up Validation

```sql
-- Must satisfy (if Positive)
followup_date BETWEEN encounter_date AND encounter_date + 2 days
```

---

## 🧪 Debug Finding (Important)

From your query:

```
Encounter Date: 2025-01-12
Screening Date: 2025-02-06 ❌
```

➡️ Outside 14-day window
➡️ Result = **NUMERATOR NOT MET**

---

## 🛠️ Fix Recommendations

### ✅ Fix 1: Ensure Correct Date Saving

* `risk_assessment_performed_on` must be:

  * NOT NULL
  * Within **measurement period (2025)**

---

### ✅ Fix 2: UI Fix (MACRA Tab)

Ensure:

* Save button correctly sets:

  ```
  risk_assessment_performed_on = encounter_date
  ```

---

### ✅ Fix 3: Backend Fix

File locations:

* `ScreeningsController.java`
* `ScreeningsServiceImpl.java`

Check:

```java
SaveScreeningResult()
SaveScreeningsquestions()
```

Ensure:

* `performed_on` is always set
* Not dependent only on UI input

---

### ✅ Fix 4: CPT Generation Issue

API returning:

```
data = "-1"
```

Check:

* `ScreeningAutoCpt` logic
* CPT 96127 mapping
* Billing configuration

---

### ✅ Fix 5: Provider Configuration

```sql
SELECT * 
FROM macra_provider_configuration
WHERE provider_id = 1 AND reporting_year = 2026;
```

Ensure:

* Reporting method = 2 (Registry / EHR)
* Measure mapped

---

## 🧩 End-to-End Flow

1. User selects PHQ-9
2. Clicks **Negative / Positive**
3. API → `SaveScreeningResult`
4. API → `SaveScreeningsquestions`
5. Data saved in:

   * `risk_assessment`
6. Measure engine evaluates:

   * Date window
   * Result code
   * Follow-up (if needed)

---

## ⚠️ Final Conclusion

👉 Measure shows **NOT MET** because:

* Screening recorded in **2026**
* Not within **2025 measurement period**
* Not within **14-day window of encounter**

---

## ✅ Expected Correct Scenario

| Field          | Value             |
| -------------- | ----------------- |
| Encounter Date | 2025-03-15        |
| Screening Date | 2025-03-10 ✅      |
| Result         | Negative          |
| Outcome        | **NUMERATOR MET** |

---
