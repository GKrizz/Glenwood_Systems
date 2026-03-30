
# 📄 Day 313 – Referral Loop (CMS50v13)

📅 **March 27, 2026 (Friday)**

---

## 📌 Measure

**Closing the Referral Loop: Receipt of Specialist Report (CMS50v13)**

---

## 🚨 Issue Summary

* For **Pfettinger account**

  * Total referrals identified: **22**
  * **3 patients missed**

    * ✅ 013108 → coming as MET
    * ❌ 3048 & 4193 → NOT coming in measure

---

## 📢 Status Update

For pfettinger account I found 22 referrals.

Out of these:

* 013108 is coming as MET ✅
* 3048 & 4193 are not coming due to issue

Root cause:
Flowsheet is not opening for these patients (UI error)

@Gobala Krishnan will fix this issue.
We will update once resolved.

---

# 👤 Patient 1: STEPHEN R. FERRUCCI III

* **Acct#:** 3048
* **Patient ID:** 7289
* **Chart ID:** 2381
* **Encounter ID:** 67255
* **Provider:** Patrick Pfettinger D.P.M
* **Encounter Date:** 2025-12-01

---

# 👤 Patient 2: WILLIAM R. SCULLY

* **Acct#:** 4193
* **Patient ID:** 5898
* **Chart ID:** 2083
* **Encounter ID:** 67325
* **Encounter Date:** 2025-12-04

---

## 📊 Referral Details

```sql id="s5j1zq"
SELECT referral_details_myalert AS patient_id,
       referral_details_chartid AS chart_id,
       referral_details_patientid AS referral_status,
       referral_order_on,
       referral_details_ord_on,
       referral_details_revdate,
       referral_receive_on,
       referral_reviewed_on
FROM referral_details
WHERE referral_details_myalert in ('7289','5898');
```

---

## 📌 Referral Status Mapping

```id="l2sk8d"
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

## 🔎 Status Verification

```sql id="1y7y7k"
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
    referral_details_revdate,
    referral_reviewed_on
FROM referral_details 
WHERE referral_details_myalert  in ('7289','5898');
```

👉 Both patients:

* Status = **Reviewed (4)**
* Referral + Review dates exist ✅

---

## ❗ Key Finding

### 🚫 No Qualifying Encounter

```sql id="6l9r4y"
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
WHERE pr.patient_registration_accountno in ('3048','4193')
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
  );
```

👉 Result: **0 rows**

---

## 🧠 Measure Logic Check

### Step 1: Qualifying Encounter ❌

### Step 2: Referral Exists ✅

### Step 3: Numerator Logic ✅

---

## 📊 Combined Validation

```sql id="2c1dhe"
WITH qualifying_encounter AS (
    SELECT DISTINCT sd.service_detail_patientid AS patient_id
    FROM service_detail sd
    JOIN cpt c ON c.cpt_id = sd.service_detail_cptid
    WHERE sd.service_detail_dos BETWEEN '2025-01-01' AND '2025-12-31'
),
first_referral AS (
    SELECT DISTINCT ON (pr.patient_registration_id)
        pr.patient_registration_id AS patient_id,
        pr.patient_registration_accountno AS account_no,
        COALESCE(rd.referral_order_on, rd.referral_details_ord_on::timestamp) AS referral_date,
        rd.referral_details_revdate AS rev_date,
        rd.referral_reviewed_on AS reviewed_on
    FROM patient_registration pr
    JOIN chart ch ON ch.chart_patientid = pr.patient_registration_id
    JOIN referral_details rd ON rd.referral_details_chartid = ch.chart_id
    WHERE pr.patient_registration_accountno in ('3048','4193')
)
SELECT
    fr.account_no,
    fr.referral_date,
    CASE WHEN qe.patient_id IS NOT NULL THEN 'YES' ELSE 'NO' END AS in_denominator,
    fr.rev_date,
    fr.reviewed_on,
    'YES' AS in_numerator
FROM first_referral fr
LEFT JOIN qualifying_encounter qe ON qe.patient_id = fr.patient_id;
```

---

## 📌 Result

| Patient | Denominator | Numerator | Final         |
| ------- | ----------- | --------- | ------------- |
| 3048    | ❌ NO        | ✅ YES     | ❌ Not counted |
| 4193    | ❌ NO        | ✅ YES     | ❌ Not counted |

---

## ⚠️ Root Cause

* Patients **DO NOT have qualifying encounter CPT codes**
* Hence:

  * ❌ Not in Initial Population
  * ❌ Not in Denominator
  * ❌ Not evaluated further

👉 Even though referral is valid → **measure will not count**

---

## 🧪 DB Validation

```sql id="9tq2dc"
SELECT * 
FROM quality_measures_patient_entries 
WHERE quality_measures_patient_entries_patient_id in ('7289','5898')
AND quality_measures_patient_entries_reporting_year = 2025
AND quality_measures_patient_entries_provider_id = 19
AND quality_measures_patient_entries_measure_id = '374';
```

👉 Result:

* IPP = 0
* DEN = 0
* NUM = 0

---

## 📊 Provider Measure Summary

```sql id="x7a0yo"
SELECT * 
FROM macra_measures_rate
WHERE macra_measures_rate_reporting_year = 2025
AND macra_measures_rate_provider_id = 19
AND macra_measures_rate_measure_id = '374';
```

👉 Provider Performance:

* ✔ IPP: 20
* ✔ DEN: 20
* ✔ NUM: 20
* ✔ Performance: **100%**

---

## 🧠 Key Learnings

* ❗ Referral alone is NOT enough
* ✅ Qualifying Encounter is mandatory
* ⚠️ If IPP = 0 → patient ignored completely
* 🔍 Always validate:

  * Encounter CPT
  * Referral
  * Dates
* 🧩 UI issues (flowsheet) can hide real logic issues

---
