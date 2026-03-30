
# 📅 Day 311 – March 25, 2026 (Wednesday)

## 🧪 CMS347v8 – Statin Therapy for the Prevention and Treatment of Cardiovascular Disease

---

## 🎯 Objective

Analyze and validate **CMS347 (Statin Therapy)** logic for patient eligibility and identify why patients are incorrectly flagged in **NOT MET** despite valid exclusion conditions.

---

## 🔍 Scenario

* Patient: **Acct# 025560 – MICHAEL MOHLER**
* Issue reported:

  * Patient flagged as **Statin Use = RED (NOT MET)**
  * Patient has **multiple statin allergies**
  * Expected behavior → Should fall under **Denominator Exception (DENEX)**

---

## 🧠 Measure Logic Overview

### Initial Population (IPP)

* Age between **40–75**
* Must have **Diabetes Diagnosis**
* Must have **Qualifying Encounter in 2025**

### Exclusions

Patient should be excluded if:

* ASCVD present
* LDL ≥ 190
* Hypercholesterolemia

### Denominator Exception (DENEX)

* Applies when **statin therapy is contraindicated** (e.g., allergies)

---

## 🧾 SQL Validation Performed

### ✅ Step 1: Qualifying Encounter Check

* CPT codes validated for 2025
* Result: **YES (Patient has qualifying encounter)**

### ✅ Step 2: ASCVD Check

* Checked via:

  * `problem_list`
  * `patient_assessments`
* Result: **NO ASCVD found**

### ⚠️ Step 3: Denominator Eligibility

* Logic in query:

  ```sql
  CASE 
    WHEN qe.patient_id IS NOT NULL AND ha.patient_id IS NOT NULL 
    THEN 'YES - Eligible for Denominator' 
    ELSE 'NO - Not Eligible'
  END
  ```
* ❌ Issue:

  * Denominator incorrectly tied to **ASCVD presence**
  * As per measure → ASCVD patients should be **excluded**, not required

### ⚠️ Step 4: Statin Exception Handling

* Statin allergy exists
* Expected: **DENEX = YES**
* Current: **NOT applied correctly**

---

## 📊 Query Result

| Account No | Patient Name    | Age | Encounter | ASCVD | Denominator | DENEX |
| ---------- | --------------- | --- | --------- | ----- | ----------- | ----- |
| 025560     | MOHLER, MICHAEL | 60  | YES       | NO    | ❌ NO        | ❌ NO  |

---

## ❗ Root Cause

1. **Incorrect Denominator Logic**

   * ASCVD used as inclusion criteria instead of exclusion
   * Violates CMS347 definition

2. **DENEX Not Triggered**

   * Statin allergy not mapped correctly to exception logic

---

## ✅ Expected Behavior

* Patient should:

  * Be included in **Initial Population**
  * Be included in **Denominator**
  * Be marked as **DENEX (Statin Allergy)**
  * NOT appear as **NOT MET**

---

## 🛠️ Fix Required

### 1. Correct Denominator Logic

* Remove ASCVD from inclusion condition
* Ensure:

  ```text
  Diabetes + Age + Encounter → Denominator
  ```

### 2. Apply ASCVD as Exclusion

* Patients with ASCVD → move to **exclusion**, not denominator filter

### 3. Fix DENEX Logic

* Map **statin allergies** properly
* Ensure:

  * Allergy → Denominator Exception

---

## 📌 Status

* 🔍 Analysis Completed
* ❗ Issue Identified (Logic + DENEX handling)
* ⏳ Fix Pending from Development Team

---

## 📧 Communication

Escalated to team via email:

> “Please address it once for all. Patient with statin allergy should not be flagged as NOT MET.”

---
