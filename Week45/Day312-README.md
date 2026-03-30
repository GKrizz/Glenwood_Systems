# 📄 Day 312 – CMS22v13 (BP Screening Measure)

📅 **March 26, 2026 (Thursday)**

---

## 📌 Measure

**CMS22v13 – Preventive Care and Screening: Screening for High Blood Pressure and Follow-Up Documented**

---

## 🚨 Issue Identified

* Many patients appearing under **❌ NOT MET**
* However, those patients already have **Hypertension (HTN)** in:

  * Problem List
  * Patient Assessment

👉 **As per CMS logic:**
These patients **should NOT be part of the measure at all (Denominator Exclusion)**

---

## 📢 Communication Sent

Hi Team,

We analyzed the issue.

The problem was that encounters qualifying for Denominator Exclusion (Hypertension diagnosis before encounter) were still being evaluated in subsequent rules.

As per measure logic, once an encounter qualifies for Denominator Exclusion, it should be removed from further evaluation and should not appear under Not Met.

We have identified the gap and are implementing a fix to ensure excluded encounters are skipped from numerator and exception evaluation.

We will update once the fix is deployed.

Thanks,
Gobala

---

## 🧠 Measure Logic Breakdown

### Initial Population

```text
Age >= 18  
AND  
Qualifying Encounter during Measurement Period
```

---

### Qualifying Encounter

```text
Encounter must be within 2025  
AND  
NOT virtual  
AND  
CPT/HCPCS/SNOMED codes from BP screening value set
```

---

### 📊 Flow Logic (IMPORTANT)

```
IPP → DEN

From DEN:
    IF matches DENEX      → REMOVE (excluded)
    ELSE IF matches NUM  → NUMERATOR
    ELSE IF DENEXCEP     → EXCEPTION
    ELSE                 → NOT MET
```

👉 **Critical Rule:**

> 🚫 **DENEX patients must NOT be evaluated further (NO numerator / NOT MET)**

---

## 📈 UI Stats Observed

| Metric  | Count |
| ------- | ----- |
| IPP     | 3336  |
| DEN     | 3336  |
| DENEX   | 2531  |
| NUM     | 647   |
| NOT MET | 158   |

👉 Issue: Some **DENEX patients leaking into NOT MET**

---

## ⚠️ Root Cause

* Logic only checking **Encounter Diagnosis**
* Missing **Problem List diagnosis**
* Missing **historical HTN before encounter**

👉 Result:
Patients incorrectly evaluated → shown as **NOT MET**

---

## 🛠️ Fix Approach

✔ Combine both sources:

* Patient Assessment
* Problem List

✔ Check:

```text
HTN diagnosis date <= Encounter date → EXCLUDE
```

---

## 🔎 Investigation Queries

### 1️⃣ Qualifying Encounters

```sql
SELECT
    pr.patient_registration_accountno AS account_no,
    pr.patient_registration_id        AS patient_id,
    sd.service_detail_id,
    sd.service_detail_dos             AS dos,
    c.cpt_cptcode
FROM patient_registration pr
JOIN service_detail sd 
    ON sd.service_detail_patientid = pr.patient_registration_id
JOIN cpt c 
    ON c.cpt_id = sd.service_detail_cptid
WHERE pr.patient_registration_accountno IN ('026803')
  AND sd.service_detail_dos BETWEEN '2025-01-01' AND '2025-12-31'
  AND c.cpt_cptcode IN (
    '90791','90792','92002','92004','92012','92014',
    '99202','99203','99204','99205','99212','99213','99214','99215',
    '99236','99281','99282','99283','99284','99285',
    '99304','99305','99306','99307','99308','99309','99310',
    '99315','99316','99341','99342','99344','99345',
    '99347','99348','99349','99350','99385','99386','99387',
    '99395','99396','99397','99424','99491'
  )
ORDER BY patient_id, dos;
```

---

### 2️⃣ Hypertension from Both Sources

```sql
SELECT
    pa.patient_assessments_patientid AS patient_id,
    pa.patient_assessments_dxcode    AS dx_code,
    pa.patient_assessments_encounterdate::date AS dx_date,
    'PATIENT ASSESSMENT' AS source
FROM patient_assessments pa
WHERE pa.patient_assessments_patientid IN (
    SELECT patient_registration_id 
    FROM patient_registration 
    WHERE patient_registration_accountno IN ('026803')
)
AND pa.patient_assessments_dxcode IN ('I10','I11.0','I11.9','I12.0','I12.9','I13.0','I13.10','I13.11','I13.2',
    'I15.0','I15.1','I15.2','I15.8','I15.9')

UNION ALL

SELECT
    pl.problem_list_patient_id       AS patient_id,
    pl.problem_list_dx_code          AS dx_code,
    COALESCE(pl.problem_list_onset_date, pl.problem_list_createdon)::date AS dx_date,
    'PROBLEM_LIST' AS source
FROM problem_list pl
WHERE pl.problem_list_patient_id IN (
    SELECT patient_registration_id 
    FROM patient_registration 
    WHERE patient_registration_accountno IN ('026803')
)
AND pl.problem_list_dx_code IN ('I10','I11.0','I11.9','I12.0','I12.9','I13.0','I13.10','I13.11','I13.2','I15.0','I15.1','I15.2','I15.8','I15.9') 
AND pl.problem_list_isactive = true;
```

---

### 3️⃣ Core Validation Logic

```sql
WITH encounters AS (
    SELECT
        pr.patient_registration_id AS patient_id,
        pr.patient_registration_accountno AS account_no,
        sd.service_detail_id,
        sd.service_detail_dos AS dos
    FROM patient_registration pr
    JOIN service_detail sd 
        ON sd.service_detail_patientid = pr.patient_registration_id
    JOIN cpt c 
        ON c.cpt_id = sd.service_detail_cptid
    WHERE pr.patient_registration_accountno IN ('026803')
      AND sd.service_detail_dos BETWEEN '2025-01-01' AND '2025-12-31'
      AND c.cpt_cptcode IN (
        '99202','99203','99204','99205','99212','99213','99214','99215'
      )
),
htn AS (
    SELECT
        pa.patient_assessments_patientid AS patient_id,
        pa.patient_assessments_encounterdate::date AS dx_date
    FROM patient_assessments pa
    WHERE pa.patient_assessments_dxcode LIKE 'I1%'

    UNION ALL

    SELECT
        pl.problem_list_patient_id AS patient_id,
        COALESCE(pl.problem_list_onset_date, pl.problem_list_createdon)::date AS dx_date
    FROM problem_list pl
    WHERE pl.problem_list_dx_code LIKE 'I1%'
      AND pl.problem_list_isactive = true
)

SELECT 
    e.account_no,
    e.patient_id,
    e.service_detail_id,
    e.dos,
    h.dx_date,
    CASE 
        WHEN h.dx_date IS NULL THEN 'NO HTN'
        WHEN h.dx_date <= e.dos THEN 'SHOULD BE EXCLUDED ✅'
        WHEN h.dx_date > e.dos THEN 'CORRECT NOT MET ✔️'
    END AS status
FROM encounters e
LEFT JOIN htn h 
    ON e.patient_id = h.patient_id
ORDER BY e.patient_id, e.dos;
```

---

### 4️⃣ Final Detailed Debug Query

```sql
WITH encounters AS (
    SELECT
        pr.patient_registration_id AS patient_id,
        pr.patient_registration_accountno AS account_no,
        sd.service_detail_id,
        sd.service_detail_dos AS dos
    FROM patient_registration pr
    JOIN service_detail sd ON sd.service_detail_patientid = pr.patient_registration_id
    JOIN cpt c ON c.cpt_id = sd.service_detail_cptid
    WHERE pr.patient_registration_accountno IN ('026803','002950','003021','003141','003196','003276','003301','003302','026116','003323','003418','028848','027004','028290','003668','025492','026838','003869','028702','024717','004352','028711','004447','004632','028439','025486','028950','027151','028744','028259','026988','024988','028316','005528','005562','028809','005763','005766','005787','028796','005915','006062','028578','027777','028546','028616','006332','028612','006405','006451','027838','028745','028945','006692','025391','006868','027825','025412','028639','028753','028708','027876','028739','0011130','007799','007842','007929','007994','008003','028951','028821','027709','028494','028850','028919','009091','028667','028792','026782','009335','028531','009366','025068','024956','028853','024793','028020','0010168','0010195','028393','0010405','0010409','028641','026151','028552','028024','0011162','024996','0011435','028383','028665','028094','026792')
      AND sd.service_detail_dos BETWEEN '2025-01-01' AND '2025-12-31'
      AND c.cpt_cptcode IN (
        '90791','90792','92002','92004','92012','92014',
        '99202','99203','99204','99205','99212','99213','99214','99215',
        '99236','99281','99282','99283','99284','99285',
        '99304','99305','99306','99307','99308','99309','99310',
        '99315','99316','99341','99342','99344','99345',
        '99347','99348','99349','99350','99385','99386','99387',
        '99395','99396','99397','99424','99491'
      )
),
htn AS (
    SELECT problem_list_patient_id AS patient_id,
           COALESCE(problem_list_onset_date, problem_list_createdon)::date AS dx_date,
           'PROBLEM_LIST' AS source
    FROM problem_list
    WHERE problem_list_dx_code IN (
        'I10','I11.0','I11.9','I12.0','I12.9','I13.0',
        'I13.10','I13.11','I13.2','I15.0','I15.1','I15.2','I15.8','I15.9'
    )
    AND problem_list_isactive = true
)

SELECT 
    e.account_no,
    e.patient_id,
    e.service_detail_id,
    e.dos,
    h.dx_date,
    h.source,
    CASE
        WHEN h.dx_date IS NULL THEN 'NO HTN FOUND'
        WHEN h.dx_date < e.dos THEN 'SHOULD BE EXCLUDED ✅'
        WHEN h.dx_date = e.dos THEN 'SAME DAY - BUG ⚠️'
        WHEN h.dx_date > e.dos THEN 'DX AFTER ENCOUNTER ✔️'
    END AS status
FROM encounters e
LEFT JOIN htn h 
    ON e.patient_id = h.patient_id
ORDER BY e.patient_id, e.dos, h.dx_date;
```

---

## ✅ Final Conclusion

* ❌ Issue: HTN patients incorrectly shown as **NOT MET**
* 🎯 Root Cause: Missing Problem List + incorrect evaluation order
* 🔧 Fix:

  * Include **Problem List + Assessment**
  * Apply **DENEX filtering first**
  * Skip further evaluation for excluded patients

---
