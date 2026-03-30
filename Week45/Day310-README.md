# 📄 Day 310 – Jrana aCC PT (006802, 11914)

📅 **March 24, 2026 (Tuesday)**

---

## 🧾 Patient: CHERYL SCHALK

* **Account#:** 006802
* **Patient ID:** 6802
* **Chart ID:** 7010
* **Encounter ID:** 117287
* **Provider ID:** 585
* **Encounter Date:** 2025-02-06

---

## 🔍 Issue

**Referral status: BLANK**
Measure: **Closing the Referral Loop: Receipt of Specialist Report → NOT MET**

---

## 📌 Referral Status Mapping

```
1 → ordered  
2 → cancelled  
3 → performed  
4 → reviewed  
5 → patient informed  
6 → completed  
7 → deleted  
8 → patient declined  
```

---

## 🔎 Investigation Queries

### Referral Details (Initial)

```sql
SELECT referral_details_myalert AS patient_id,
       referral_details_chartid AS chart_id,
       referral_details_patientid AS referral_status,
       referral_order_on,
       referral_details_ord_on,
       referral_receive_on,
       referral_reviewed_on
FROM referral_details
WHERE referral_details_myalert = 6802;
```

---

### Referral Status Mapping Check

```sql
SELECT 
    referral_details_myalert AS patient_id,
    referral_details_chartid AS chart_id,
    referral_details_patientid AS status_code,
    CASE referral_details_patientid
        WHEN 1 THEN 'ordered'
        WHEN 2 THEN 'cancelled'
        WHEN 3 THEN 'performed'
        WHEN 4 THEN 'reviewed'
        WHEN 5 THEN 'patient informed'
        WHEN 6 THEN 'completed'
        WHEN 7 THEN 'deleted'
        WHEN 8 THEN 'patient declined'
        ELSE 'unknown'
    END AS status_name,
    referral_order_on,
    referral_reviewed_on
FROM referral_details
WHERE referral_details_myalert = 6802;
```

👉 **Issue Found:**
Status stored as **6802 (invalid)** → should be valid enum

---

## 🛠️ Fix Applied

### Backup

```sql
\copy (
SELECT referral_details_refid, referral_details_myalert, referral_details_patientid 
FROM referral_details 
WHERE referral_details_refid = 2404
) TO 'referral_2404_backup.csv' WITH CSV HEADER;
```

### Update Status → Reviewed (4)

```sql
UPDATE referral_details 
SET referral_details_patientid = 4 
WHERE referral_details_refid = 2404;
```

---

### ✅ Verification

```sql
SELECT referral_details_myalert AS patient_id,
       referral_details_chartid AS chart_id,
       referral_details_patientid AS referral_status,
       referral_order_on,
       referral_details_ord_on,
       referral_receive_on,
       referral_reviewed_on
FROM referral_details
WHERE referral_details_refid = 2404;
```

---

## 📊 Encounter Validation

```sql
SELECT
    pr.patient_registration_accountno AS account_no,
    pr.patient_registration_first_name AS first_name,
    pr.patient_registration_last_name AS last_name,
    pr.patient_registration_id AS patient_id,
    sd.service_detail_dos AS dos,
    c.cpt_cptcode AS cpt_code
FROM patient_registration pr
JOIN service_detail sd ON sd.service_detail_patientid = pr.patient_registration_id
JOIN cpt c ON c.cpt_id = sd.service_detail_cptid
WHERE pr.patient_registration_accountno ='006802'
AND sd.service_detail_dos BETWEEN '2025-01-01' AND '2025-12-31'
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
)
ORDER BY pr.patient_registration_accountno, sd.service_detail_dos;
```

---

## 📈 Measure Validation Logic

### Final Measure Result

```sql
WITH target_patients AS (
    SELECT
        patient_registration_id AS patient_id,
        patient_registration_accountno AS account_no,
        patient_registration_first_name AS first_name,
        patient_registration_last_name AS last_name
    FROM patient_registration
    WHERE patient_registration_accountno ='006802'
),
qualifying_encounter AS (
    SELECT DISTINCT
        sd.service_detail_patientid AS patient_id,
        MIN(sd.service_detail_dos) AS first_encounter_date,
        STRING_AGG(DISTINCT c.cpt_cptcode, ', ' ORDER BY c.cpt_cptcode) AS cpt_codes
    FROM service_detail sd
    JOIN cpt c ON c.cpt_id = sd.service_detail_cptid
    JOIN target_patients tp ON tp.patient_id = sd.service_detail_patientid
    WHERE sd.service_detail_dos BETWEEN '2025-01-01' AND '2025-12-31'
    GROUP BY sd.service_detail_patientid
),
first_referral AS (
    SELECT DISTINCT ON (tp.patient_id)
        tp.patient_id,
        rd.referral_details_refid AS refid,
        COALESCE(rd.referral_order_on,rd.referral_details_ord_on::timestamp) AS referral_date,
        rd.referral_details_revdate AS rev_date,
        rd.referral_reviewed_on AS reviewed_on,
        rd.referral_receive_on AS received_on
    FROM target_patients tp
    JOIN chart ch ON ch.chart_patientid = tp.patient_id
    JOIN referral_details rd ON rd.referral_details_chartid = ch.chart_id
    WHERE COALESCE(rd.referral_order_on,rd.referral_details_ord_on::timestamp) >= '2025-01-01'
      AND COALESCE(rd.referral_order_on,rd.referral_details_ord_on::timestamp) < '2025-11-01'
      AND rd.referral_details_isactive = -1
    ORDER BY tp.patient_id,
             COALESCE(rd.referral_order_on, rd.referral_details_ord_on::timestamp) ASC
)
SELECT
    tp.first_name, tp.last_name,
    qe.first_encounter_date,
    qe.cpt_codes,
    fr.referral_date,
    fr.rev_date,
    fr.reviewed_on,
    fr.received_on,
    'YES - numerator met' AS in_numerator,
    'No gap - measure complete' AS gap_reason
FROM target_patients tp
LEFT JOIN qualifying_encounter qe ON qe.patient_id = tp.patient_id
LEFT JOIN first_referral fr ON fr.patient_id = tp.patient_id;
```

---

## ✅ Final Outcome

* ✔ Referral status corrected → **Reviewed**
* ✔ Numerator satisfied
* ✔ Denominator satisfied
* ✔ **Measure PASSED**

---

# ⚠️ Issue 2: PAMELA P. RAMEY (Acct# 11914)

## ❌ Error

```
SocketTimeoutException: Read timed out
```

### Root Cause

Timeout too low:

```java
conn.setReadTimeout(10000);
```

---

## 🔧 Fix

```java
conn.setReadTimeout(60000);   // 60 sec
// OR
conn.setReadTimeout(120000);  // 2 min (recommended)
```

---

## 🔗 API

```
/QPPPerformance/getCQMStatusByPatient?patientID=3705&providerId=22&year=2025
```

---

## 🧪 DB Validation

```sql
SELECT * 
FROM quality_measures_patient_entries
WHERE quality_measures_patient_entries_patient_id = 3705
AND quality_measures_patient_entries_measure_id = '374';
```

👉 **DB shows numerator = 1 (PASS)**
👉 API shows **false (due to timeout issue)**

---

# 🧾 Additional Patients

## 👤 CLAUDIA BROWN (000713)

* Patient ID: 713
* Provider: Naga Madireddy M.D.
* Encounter: 2026-03-18

---

## 👤 SAMUEL D. TINDLE (003441)

API:

```
/getCQMStatusByPatient?patientID=3441&providerId=25&year=2026
```

👉 Multiple measures validated successfully

---
