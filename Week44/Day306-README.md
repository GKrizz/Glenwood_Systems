
# 📘 README: Auto-Population of CPT II Codes (Case #243527 – MIM)

---

# 🎯 Objective

Automatically generate **CPT II Codes (Quality Codes)** based on clinical data:

| Activity             | Example CPT II        |
| -------------------- | --------------------- |
| Depression Screening | 3725F                 |
| Blood Pressure       | 3074F / 3075F / 3077F |
| Hemoglobin A1c       | 3044F / 3045F / 3046F |
| Colonoscopy          | 3017F                 |
| Mammogram            | 3014F                 |

---

# 🚨 Current Problem

```text
CPT II codes are NOT auto-generated ❌
Manual entry required ❌
```

👉 Impact:

* Missed MIPS measures
* Incomplete reporting
* Workflow inefficiency

---

# 🧠 System Flow (Expected)

```text
Clinical Data Entry
   ↓
patient_clinical_elements / labs / imaging
   ↓
Rule Engine (CPT II Mapping)
   ↓
service_detail (CPT II auto insert)
   ↓
Billing / Reporting / MIPS
```

---

# 🔍 Root Cause

### 🔴 Missing Automation Layer

Currently:

* Data exists in:

  * `patient_clinical_elements`
  * `lab_results`
  * `risk_assessment`
* BUT ❌ no logic to generate CPT II

---

# 🧩 DATA SOURCES

| Activity             | Table                              |
| -------------------- | ---------------------------------- |
| Depression Screening | patient_clinical_elements          |
| BP                   | vitals / patient_clinical_elements |
| HbA1c                | lab_results                        |
| Colonoscopy          | procedure / history                |
| Mammogram            | imaging                            |

---

# 🛠️ SOLUTION DESIGN

---

# ✅ STEP 1: CPT II Mapping Logic

---

## 🔹 Depression Screening

```sql
SELECT patient_id, encounter_id
FROM patient_clinical_elements
WHERE gwid IN ('PHQ9_GWID')
AND value IS NOT NULL;
```

➡ Insert:

```sql
INSERT INTO service_detail (patient_id, cpt_code, dos)
VALUES (<patient_id>, '3725F', CURRENT_DATE);
```

---

## 🔹 Blood Pressure

```sql
SELECT systolic, diastolic
FROM vitals
WHERE patient_id = <id>;
```

### Mapping:

| Condition       | CPT II |
| --------------- | ------ |
| <120/80         | 3074F  |
| 120–139 / 80–89 | 3075F  |
| ≥140 / ≥90      | 3077F  |

```sql
CASE 
WHEN systolic < 120 AND diastolic < 80 THEN '3074F'
WHEN systolic BETWEEN 120 AND 139 THEN '3075F'
ELSE '3077F'
END
```

---

## 🔹 HbA1c

```sql
SELECT result_value
FROM lab_results
WHERE loinc = '4548-4';
```

### Mapping:

| HbA1c | CPT II |
| ----- | ------ |
| <7    | 3044F  |
| 7–9   | 3045F  |
| >9    | 3046F  |

---

## 🔹 Colonoscopy

```sql
SELECT *
FROM procedure_history
WHERE procedure_code IN ('colonoscopy_codes');
```

➡ CPT II: `3017F`

---

## 🔹 Mammogram

```sql
SELECT *
FROM imaging
WHERE type = 'Mammogram';
```

➡ CPT II: `3014F`

---

# 🧩 STEP 2: AUTO INSERT ENGINE

---

## 🔧 Core Insert Logic

```sql
INSERT INTO service_detail (
    service_detail_patientid,
    service_detail_cptid,
    service_detail_dos
)
SELECT
    p.patient_id,
    c.cpt_id,
    CURRENT_DATE
FROM temp_cptii_mapping p
JOIN cpt c ON c.cpt_cptcode = p.cpt_code
WHERE NOT EXISTS (
    SELECT 1 FROM service_detail sd
    WHERE sd.patient_id = p.patient_id
    AND sd.cpt_id = c.cpt_id
);
```

---

# 🧠 STEP 3: TRIGGER / JOB OPTIONS

---

## ✅ Option 1: Real-Time Trigger

```sql
AFTER INSERT ON patient_clinical_elements
```

👉 Pros: instant
👉 Cons: performance

---

## ✅ Option 2: Batch Job (Recommended)

```text
Run nightly job:
generate_cptii_codes()
```

👉 Pros: scalable
👉 Cons: slight delay

---

# 🧪 VALIDATION

---

## 🔍 Check Generated CPT II

```sql
SELECT *
FROM service_detail
WHERE cpt_code IN ('3725F','3074F','3044F','3017F','3014F')
AND patient_id = <id>;
```

---

## 🔍 Prevent Duplicates

```sql
SELECT patient_id, cpt_code, COUNT(*)
FROM service_detail
GROUP BY patient_id, cpt_code
HAVING COUNT(*) > 1;
```

---

# ⚠️ EDGE CASES

---

| Scenario             | Handling                 |
| -------------------- | ------------------------ |
| Multiple BP readings | Use latest               |
| Multiple HbA1c       | Use latest               |
| Missing encounter    | link to latest encounter |
| Duplicate CPT II     | prevent via NOT EXISTS   |

---

# 🛡️ PREVENTIVE DESIGN

---

## 🔴 Add Flag Column

```sql
is_auto_generated = true
```

---

## 🔴 Logging

```sql
INSERT INTO audit_log (event)
VALUES ('CPT II auto-generated');
```

---

## 🔴 Config Table (Flexible)

```sql
cptii_mapping_config
```

| activity | gwid | cpt_code |
| -------- | ---- | -------- |

---

# 🏁 FINAL SUMMARY

| Problem                   | Fix                  |
| ------------------------- | -------------------- |
| CPT II not auto-generated | Add rule engine      |
| Manual entry              | Automate via SQL/job |
| Missing reporting         | Auto-populate        |

---




# 📘 README: Measure 374 – Closing the Referral Loop (Case #243527)

---

# 🎯 Objective

Fix why **Patient 3295672** is showing:

```text
❌ NOT MET (Numerator)
```

Even though:

* Referral exists ✅
* Reviewed / Received dates exist ✅

---

# 🧠 Measure Logic (CMS 374 Simplified)

---

## ✅ Denominator

Patient must have:

1. **Qualifying Encounter (CPT codes)**
2. **Referral between Jan–Oct 2025**

✔ Your data:
ALL 3 patients → ✅ IN DENOMINATOR

---

## ✅ Numerator (CRITICAL)

A referral is **COUNTED only if**:

```text
Specialist report is RECEIVED or REVIEWED
AFTER referral date
WITHIN measurement year (2025)
```

---

# 🔍 ROOT CAUSE (Your Case)

---

## 🔴 Patient: 3295672

| Field         | Value        |
| ------------- | ------------ |
| Referral Date | 2025-02-21 ✅ |
| Rev Date      | 2025-02-22 ✅ |
| Reviewed On   | ❌ 2026-03-18 |
| Received On   | ❌ 2026-03-18 |

---

## ⚠️ Problem

```text
System is using: reviewed_on / received_on ❌
Instead of: rev_date ✅
```

👉 Because:

```text
reviewed_on = 2026 ❌ (outside measurement year)
received_on = 2026 ❌
```

---

# 🚨 Why It Shows NOT MET

Your logic:

```sql
COALESCE(rev_date, reviewed_on)
```

BUT actual system logic likely:

```sql
COALESCE(received_on, reviewed_on)
```

👉 So it evaluates:

```text
2026 date → outside 2025 → ❌ NOT MET
```

---

# 🔬 Proof from Your Data

---

## ✅ Working Patients

They pass because:

* System probably picked **rev_date (2025)**

---

## ❌ Failing Patient (3295672)

Even though:

* rev_date = 2025 ✅

System is picking:

* reviewed_on = 2026 ❌

---

# 🛠️ FIX OPTIONS

---

# ✅ FIX 1: Backend Logic Correction (RECOMMENDED)

Change numerator logic to:

```sql
COALESCE(
    rd.referral_details_revdate,
    rd.referral_reviewed_on,
    rd.referral_receive_on
)
```

✔ Priority:

1. rev_date (best clinical indicator)
2. reviewed_on
3. received_on

---

# ✅ FIX 2: Enforce Measurement Year Filter

```sql
WHERE COALESCE(date_field) BETWEEN '2025-01-01' AND '2025-12-31'
```

---

# ✅ FIX 3: Data Fix (Quick Fix)

Update reviewed_on to 2025:

```sql
UPDATE referral_details
SET referral_reviewed_on = referral_details_revdate
WHERE referral_details_refid = <refid>;
```

---

# ✅ FIX 4: Debug Query (Final Truth)

Run this to confirm what system is using:

```sql
SELECT
    referral_details_refid,
    referral_details_revdate,
    referral_reviewed_on,
    referral_receive_on,

    COALESCE(referral_receive_on, referral_reviewed_on) AS system_used_date,

    CASE 
        WHEN COALESCE(referral_receive_on, referral_reviewed_on)
             BETWEEN '2025-01-01' AND '2025-12-31'
        THEN 'NUMERATOR'
        ELSE 'NOT MET'
    END AS result
FROM referral_details
WHERE referral_details_chartid = 115147;
```

---

# 🧪 Expected Result After Fix

```text
Patient 3295672 → ✅ NUMERATOR MET
```

---

# ⚠️ SECOND ISSUE (IMPORTANT)

---

## 🚨 API Returning:

```json
"accountId": "",
"patientId": 0
```

👉 This means:

```text
❌ Context issue (same as previous cases)
```

---

## 🔴 Root Cause

* Wrong providerId OR
* userId mismatch OR
* session issue

---

## ✅ Fix

Ensure:

```text
providerId = 12 (correct rendering provider)
userId = same as provider or logged user
accountId = drc
```

---

# 🏁 FINAL SUMMARY

---

| Issue                   | Root Cause                    | Fix                |
| ----------------------- | ----------------------------- | ------------------ |
| Patient 3295672 NOT MET | System using 2026 reviewed_on | Use rev_date first |
| Numerator mismatch      | Wrong COALESCE priority       | Fix backend logic  |
| eCQM not loading        | Context mismatch              | Fix provider/user  |

---
