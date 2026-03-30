
# 📅 Day 309 – March 23, 2026 (Monday)

## 🩺 Preventive Care and Screening: Screening for High Blood Pressure and Follow-Up Documented

---

## 🎯 Objective

Analyze **BP Screening (CMS165-like measure)** logic to ensure:

* Correct **Initial Population & Denominator inclusion**
* Proper handling of **Hypertension exclusions**
* Accurate **Numerator compliance based on BP readings + interventions**

---

## 🔍 Measure Overview

### ✅ Initial Population (IPP)

Patient must:

* Be **≥ 18 years** at start of Measurement Period
* Have at least **one qualifying encounter** during 2025

### 📌 Qualifying Encounters

Includes CPTs such as:

* Office Visits: `99202–99215`
* Preventive Visits: `99385–99397`
* ER / Other visits: `99281–99285`
* Eye exams, psych evals, etc.

---

## 🚫 Denominator Exclusions

Patient is excluded if:

* Has **Hypertension diagnosis**
* Diagnosis overlaps **before or during encounter**

### Key Codes:

* ICD-10: `I10, I11.*, I12.*, I13.*, I15.*, I27.*`
* Includes:

  * Essential HTN
  * Secondary HTN
  * Pulmonary HTN

---

## 🧠 Numerator Logic (Complex)

Patient meets numerator if **ANY one** of the following is satisfied:

---

### 🟢 1. Normal BP Reading

* BP documented and within normal range

---

### 🟡 2. Elevated BP (SBP 120–129 AND DBP <80)

Must include:

* Follow-up within **6 months**
* AND lifestyle/non-pharmacological intervention
  **OR**
* Referral to another provider

---

### 🔴 3. First Hypertensive Reading (≥130 / ≥80)

Requires:

* Intervention OR referral on same day

---

### 🟠 4. Second Hypertensive Reading (130–139 / 80–89)

Requires:

* Follow-up within 6 months
* Lab/ECG
* Lifestyle intervention
  **OR**
* Referral

---

### 🔴 5. Severe Hypertension (≥140 / ≥90)

Requires:

* Follow-up within **4 weeks**
* Lab/ECG
* Lifestyle intervention
* Medication
  **OR**
* Referral

---

## 🔧 Key Data Points Used

### 🩸 Blood Pressure

* **Systolic (LOINC 8480-6)**
* **Diastolic (LOINC 8462-4)**
* Uses **last reading during encounter**

### 🧾 Interventions

* Lifestyle recommendations
* Diet / weight reduction
* Physical activity
* Alcohol counseling

### 📤 Referrals

* Referral to PCP / specialist
* Must include reason: **Elevated BP / Hypertension**

---

## ❗ Observations

1. **Hypertension Exclusion Critical**

   * Any HTN diagnosis removes patient from denominator
   * Must validate diagnosis timing carefully

2. **Strict Same-Day Logic**

   * Interventions must occur **on same day as encounter**
   * Missing timestamp → leads to **NOT MET**

3. **Last BP Reading Used**

   * Only latest BP during encounter considered
   * Earlier readings ignored

4. **Complex Multi-Path Numerator**

   * Multiple pathways → increases risk of logic gaps

---

## ⚠️ Potential Issues Identified

* Missing **BP recordings** → patient fails numerator
* Intervention not tied to **same encounter date**
* Referral without proper **reason code**
* Incorrect handling of **prior HTN diagnosis**

---

## 🛠️ Recommendations

1. **Validate BP Capture**

   * Ensure both systolic & diastolic recorded

2. **Fix Timestamp Alignment**

   * Interventions must match encounter date

3. **Improve HTN Diagnosis Logic**

   * Ensure correct overlap logic (not over-excluding)

4. **Enhance Data Mapping**

   * Map all intervention value sets properly

---

## 📌 Status

* 🔍 Measure logic fully reviewed
* ⚠️ Multiple edge cases identified
* ⏳ Requires validation with real patient data

---
