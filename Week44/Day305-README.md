
# 📘 README: CMS50v13 – Referral Loop Closure (MIPS)

### 📅 Reporting Year: 2025

---

# 🎯 Objective

Evaluate and fix **Referral Loop Closure Measure (CMS50v13)**

This measure checks:

> ✅ Patient had qualifying encounter
> ✅ Referral was made (Jan–Oct)
> ✅ Referral result was **received AFTER referral order date**

---

# 📊 Patients Covered

### 🔹 Set 1

* 3242928 (LINDA ABNER)
* 3271530 (JAMES ABSTON)
* 3295772 (CORA ADAMS)

### 🔹 Set 2

* 3302337, 3303394, 3303725
* 3303937, 3295675, 3308327, 3311345

---

# 🧠 Measure Logic (Simplified)

```text
IPP → Qualifying Encounter (2025)

DENOM → Referral exists (Jan–Oct 2025)

NUMERATOR → Referral reviewed/received AFTER referral date
```

---

# 🧩 STEP 1: Qualifying Encounter

```sql
SELECT DISTINCT sd.service_detail_patientid AS patient_id
FROM service_detail sd
JOIN cpt c ON c.cpt_id = sd.service_detail_cptid
WHERE sd.service_detail_dos BETWEEN '2025-01-01' AND '2025-12-31'
AND c.cpt_cptcode IN (
'99202','99203','99204','99205',
'99212','99213','99214','99215',
'92002','92004','92012','92014',
'99381','99382','99383','99384',
'99385','99386','99387',
'99395','99396','99397',
'99391','99392','99393','99394',
'96116','96156','96136','96138',
'90839','90791','90792','96112'
);
```

✅ Result:
All 3 patients **HAVE qualifying encounters**

---

# 🧩 STEP 2: First Referral (Jan–Oct 2025)

```sql
SELECT DISTINCT ON (pr.patient_registration_id)
    pr.patient_registration_id AS patient_id,
    COALESCE(rd.referral_order_on, rd.referral_details_ord_on::timestamp) AS referral_date
FROM patient_registration pr
JOIN chart ch ON ch.chart_patientid = pr.patient_registration_id
JOIN referral_details rd ON rd.referral_details_chartid = ch.chart_id
WHERE pr.patient_registration_accountno IN ('3242928','3271530','3295772')
AND COALESCE(rd.referral_order_on, rd.referral_details_ord_on::timestamp)
    BETWEEN '2025-01-01' AND '2025-10-31'
ORDER BY pr.patient_registration_id, referral_date;
```

---

# 🧩 STEP 3: Numerator Logic (CRITICAL)

### ✔ Rule:

```text
received_date > referral_order_date
```

---

# ❌ CURRENT ISSUE (ROOT CAUSE)

### 🔴 All patients failing numerator

```text
email_received_date < referral_order_date ❌
```

### Example:

| Patient | Referral Date | Email Received | Result   |
| ------- | ------------- | -------------- | -------- |
| 3302337 | Apr 9         | Jan 13         | ❌ BEFORE |
| 3303937 | Jun 4         | Feb 25         | ❌ BEFORE |
| 3311345 | Oct 24        | Oct 10         | ❌ BEFORE |

---

# 🚨 Root Cause

👉 System is picking **WRONG email**

```text
Picking EARLIEST email ❌
Instead of FIRST VALID email AFTER referral ✅
```

---

# 🛠️ CORRECT FIX (IMPORTANT)

---

## ✅ Fix Query (Correct Logic)

```sql
WITH referral_data AS (
    SELECT DISTINCT ON (pr.patient_registration_id)
        pr.patient_registration_id AS patient_id,
        COALESCE(rd.referral_details_ord_on, rd.referral_order_on) AS referral_date
    FROM patient_registration pr
    JOIN chart ch ON ch.chart_patientid = pr.patient_registration_id
    JOIN referral_details rd ON rd.referral_details_chartid = ch.chart_id
    WHERE pr.patient_registration_accountno IN (
        '3302337','3303394','3303725','3303937',
        '3295675','3308327','3311345'
    )
    AND COALESCE(rd.referral_details_ord_on, rd.referral_order_on)
        BETWEEN '2025-01-01' AND '2025-10-31'
    ORDER BY pr.patient_registration_id, referral_date
),

valid_email AS (
    SELECT
        ir.incoming_referral_patient_id AS patient_id,
        dw.document_workflow_sent_on AS received_date,
        ROW_NUMBER() OVER (
            PARTITION BY ir.incoming_referral_patient_id
            ORDER BY dw.document_workflow_sent_on
        ) AS rn
    FROM incoming_referral ir
    JOIN document_workflow dw
        ON ir.incoming_referral_document_workflow_id = dw.document_workflow_id
    WHERE dw.document_workflow_sent_on >= '2025-01-01'
),

matched_email AS (
    SELECT r.patient_id, r.referral_date, v.received_date
    FROM referral_data r
    JOIN valid_email v
        ON v.patient_id = r.patient_id
    WHERE v.received_date > r.referral_date
)

SELECT
    r.patient_id,
    r.referral_date,
    MIN(m.received_date) AS valid_received_date,
    CASE 
        WHEN MIN(m.received_date) IS NOT NULL THEN 'YES'
        ELSE 'NO'
    END AS meets_numerator
FROM referral_data r
LEFT JOIN matched_email m
    ON r.patient_id = m.patient_id
GROUP BY r.patient_id, r.referral_date
ORDER BY r.patient_id;
```

---

# 🎯 Expected Behavior After Fix

| Scenario                | Result   |
| ----------------------- | -------- |
| Email AFTER referral    | ✅ PASS   |
| Email BEFORE referral   | ❌ IGNORE |
| No email after referral | ❌ FAIL   |

---

# ⚠️ Common Mistakes (VERY IMPORTANT)

---

## ❌ Wrong Logic

```sql
ORDER BY email_date ASC
```

👉 Picks oldest email → WRONG

---

## ✅ Correct Logic

```sql
WHERE email_date > referral_date
ORDER BY email_date ASC
LIMIT 1
```

👉 Picks **first valid email after referral**

---

# 🔄 Alternative (Using referral_details table)

If using:

```sql
referral_receive_on
referral_reviewed_on
```

Then fix:

```sql
CASE 
WHEN referral_receive_on > referral_order_date THEN 'YES'
ELSE 'NO'
END
```

---

# 🧪 Validation Query

```sql
SELECT
referral_order_date,
referral_receive_on,
CASE 
WHEN referral_receive_on > referral_order_date THEN 'VALID'
ELSE 'INVALID'
END
FROM referral_details;
```

---

# 🛡️ Preventive Fix

---

## 🔴 Backend Fix

```java
// WRONG
pickEarliestEmail();

// CORRECT
pickFirstEmailAfterReferral();
```

---

## 🔴 Add Logging

```java
log.info("Referral Date: " + referralDate);
log.info("Selected Email Date: " + emailDate);
```

---

## 🔴 UI Fix

* Show:

  * Referral Date
  * Received Date
  * Status (Valid / Invalid)

---

# 🏁 Final Summary

| Step      | Status          |
| --------- | --------------- |
| Encounter | ✅               |
| Referral  | ✅               |
| Numerator | ❌ (logic issue) |

---


# 📘 README: Depression Remission at 12 Months (CMS159v13)

### 📅 Case #241929 | Reporting Year: 2025

---

# 🎯 Objective

Evaluate **Depression Remission at 12 Months**

> ✔ Patient had PHQ-9 > 9 (Index)
> ✔ Follow-up PHQ-9 in 12 months
> ✔ Latest PHQ-9 score < 5 → ✅ REMISSION

---

# 🧠 Measure Flow (Simplified)

```text id="p3k7zn"}
Index PHQ-9 (>9)
     ↓
Within Encounter Window
     ↓
12-Month Follow-Up Period
     ↓
Last PHQ-9 Score < 5 → NUMERATOR ✅
```

---

# 📊 Patients Analyzed

| Account | Patient ID | Age |
| ------- | ---------- | --- |
| 006780  | 6780       | 55  |
| 014747  | 14747      | 20  |
| 015153  | 15153      | 46  |
| 015309  | 15309      | 36  |

---

# 🧩 STEP 1: Age Check (IPP)

```sql id="z8i3mp"}
SELECT patient_registration_id,
DATE_PART('year', AGE('2025-01-01', patient_registration_dob)) AS age
FROM patient_registration
WHERE patient_registration_accountno IN ('006780','014747','015309','015153');
```

✅ All patients ≥ 12 → INCLUDED

---

# 🧩 STEP 2: Identify Qualifying Encounter (IMPORTANT WINDOW)

### 📌 Denominator Identification Period

```text id="fxd2oj"}
2023-11-01 → 2024-11-01
```

```sql id="3g2jlf"}
SELECT sd.service_detail_id, sd.service_detail_dos, c.cpt_cptcode
FROM service_detail sd
JOIN cpt c ON c.cpt_id = sd.service_detail_cptid
WHERE sd.service_detail_patientid IN (6780,14747,15153,15309)
AND sd.service_detail_dos BETWEEN '2023-11-01' AND '2024-11-01'
AND c.cpt_cptcode IN (
'90791','90792','90832','90834','90837',
'99202','99203','99204','99205',
'99211','99212','99213','99214','99215'
);
```

✅ Encounters found → PASS

---

# 🧩 STEP 3: Depression Diagnosis

```sql id="g6m1xk"}
SELECT patient_id, dx_code
FROM (
   SELECT patient_assessments_patientid AS patient_id,
          patient_assessments_dxcode AS dx_code
   FROM patient_assessments
   UNION
   SELECT problem_list_patient_id,
          problem_list_dx_code
   FROM problem_list
) t
WHERE dx_code IN ('F32.x','F33.x','F34.1');
```

✅ Diagnosis present → PASS

---

# 🧩 STEP 4: INDEX PHQ-9 (CRITICAL)

### ✔ First PHQ-9 > 9

```sql id="o1n4zy"}
SELECT
pce.patient_clinical_elements_patientid AS patient_id,
pce.patient_clinical_elements_created_on AS phq9_date,
pce.patient_clinical_elements_value AS score
FROM patient_clinical_elements pce
JOIN cnm_code_system ccs
ON ccs.cnm_code_system_gwid = pce.patient_clinical_elements_gwid
WHERE ccs.cnm_code_system_code IN ('44261-6','89204-2')
AND pce.patient_clinical_elements_value::int > 9
ORDER BY phq9_date;
```

---

# ⚠️ Common Failure (ROOT CAUSE)

👉 Missing OR incorrect index:

```text id="bn4y6s"}
No PHQ-9 > 9 found ❌
OR
Not linked to encounter ❌
```

---

# 🧩 STEP 5: NUMERATOR (REMISSION CHECK)

### ✔ Last PHQ-9 in 2025

```sql id="7l8yhp"}
SELECT
pce.patient_clinical_elements_patientid,
MAX(pce.patient_clinical_elements_created_on) AS last_phq9_date,
LAST_VALUE(pce.patient_clinical_elements_value::int)
OVER (PARTITION BY pce.patient_clinical_elements_patientid
ORDER BY pce.patient_clinical_elements_created_on
ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING) AS last_score
FROM patient_clinical_elements pce
JOIN cnm_code_system ccs
ON ccs.cnm_code_system_gwid = pce.patient_clinical_elements_gwid
WHERE ccs.cnm_code_system_code IN ('44261-6','89204-2')
AND pce.patient_clinical_elements_created_on BETWEEN '2025-01-01' AND '2025-12-31';
```

---

# 🎯 Numerator Logic

```sql id="d6q2lf"}
CASE 
WHEN last_score < 5 THEN 'MET ✅'
ELSE 'NOT MET ❌'
END
```

---

# 🚨 ROOT CAUSE (Based on Your Data)

### 🔴 Issue Identified:

```text id="s2a4ld"}
PHQ-9 recorded dates NOT within correct window
OR
Index PHQ-9 not properly identified
OR
Latest PHQ-9 ≥ 5
```

---

# 🛠️ FIX OPTIONS

---

## ✅ FIX 1: Ensure Proper Index PHQ-9

```sql id="m9r3zc"}
-- Insert or update correct PHQ-9 (>9) in 2024
```

✔ Must be:

* > 9
* Within encounter window

---

## ✅ FIX 2: Add Follow-Up PHQ-9 in 2025

```sql id="7ck0ys"}
-- Ensure PHQ-9 exists in 2025
-- Score must be < 5
```

---

## ✅ FIX 3: Link to Encounter

```sql id="v2n9fw"}
UPDATE patient_clinical_elements
SET patient_clinical_elements_encounterid = <valid_encounter_id>
WHERE patient_clinical_elements_id = <id>;
```

---

# 🧪 FINAL VALIDATION QUERY

```sql id="f0k8xm"}
WITH phq AS (
SELECT
patient_id,
created_on,
value::int AS score
FROM patient_clinical_elements
)
SELECT
patient_id,
MIN(CASE WHEN score > 9 THEN created_on END) AS index_date,
MAX(CASE WHEN created_on BETWEEN '2025-01-01' AND '2025-12-31'
         THEN score END) AS last_score
FROM phq
GROUP BY patient_id;
```

---

# 🛡️ Preventive Fix

---

## 🔴 Common Issues

| Issue                     | Impact         |
| ------------------------- | -------------- |
| PHQ-9 not mapped to LOINC | Measure fails  |
| No encounter linkage      | Index invalid  |
| Wrong dates               | Outside window |

---

## ✅ Best Practice

```text id="t6x9ph"}
Always ensure:
1. PHQ-9 > 9 (index)
2. Linked to valid encounter
3. Follow-up PHQ-9 in next 12 months
4. Final score < 5
```

---

# 🏁 Final Summary

| Step            | Status   |
| --------------- | -------- |
| Age             | ✅        |
| Encounter       | ✅        |
| Diagnosis       | ✅        |
| Index PHQ-9     | ⚠️ Check |
| Follow-up PHQ-9 | ⚠️ Check |

---
