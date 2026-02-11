# CMS138v14 (Measure 226) – Smoking Status Not Reflecting in MIPS

**Account:** AKM (Awani Kumar, MD, PC)
**Case #:** 241552
**Reported By:** Dr. Awani Kumar
**Assigned To:** Gobala Krishnan
**Year:** 2026

---

# 1️⃣ Issue Summary

### ❗ Problem Reported

* Smoking status updated via **MIPS Flowsheet**
* UI shows **“Saved Successfully”**
* But MIPS Measure 226 (CMS138v14) still shows **“Not Met”**
* Worked fine in other accounts

---

# 2️⃣ Measure Involved

### CMS138v14

**Preventive Care and Screening: Tobacco Use: Screening and Cessation Intervention**

Measure requires:

### ✔ Initial Population

* Age ≥ 12 at start of Measurement Period
* ≥ 2 qualifying visits OR 1 preventive visit

### ✔ Denominator

* Same as Initial Population

### ✔ Numerator

* Tobacco screening documented
  AND
* Cessation intervention documented (if tobacco user)

---

# 3️⃣ Initial Data Verification

We verified:

## ✅ Patient Age

```sql
SELECT
  patient_registration_dob,
  EXTRACT(YEAR FROM AGE('2026-01-01', patient_registration_dob)) AS age
FROM patient_registration
WHERE patient_registration_id = 18439;
```

✔ Age = 84 → Meets IPP age condition

---

## ✅ Qualifying Visits Check

```sql
SELECT
    sd.service_detail_id,
    c.cpt_cptcode,
    sd.service_detail_dos
FROM service_detail sd
JOIN cpt c ON c.cpt_id = sd.service_detail_cptid
WHERE sd.service_detail_patientid = 18439
  AND sd.service_detail_dos BETWEEN '2026-01-01' AND '2026-12-31'
  AND c.cpt_cptcode IN (list_of_qualifying_codes);
```

✔ 2 qualifying visits found
✔ IPP & DENOM satisfied

---

# 4️⃣ Smoking Status Storage Verification

## Clinical Element Storage

```sql
SELECT
  pce.patient_clinical_elements_gwid,
  pce.patient_clinical_elements_value,
  ceo.clinical_elements_options_name,
  ceo.clinical_elements_options_snomed
FROM patient_clinical_elements pce
LEFT JOIN clinical_elements_options ceo
  ON ceo.clinical_elements_options_gwid  = pce.patient_clinical_elements_gwid
 AND ceo.clinical_elements_options_value = pce.patient_clinical_elements_value
WHERE pce.patient_clinical_elements_patientid = 18439
  AND pce.patient_clinical_elements_encounterid = 13875;
```

✔ Smoking status saved
✔ SNOMED = 77176002 (Smoker)

So data WAS saving correctly in DB.

---

# 5️⃣ Intervention Verification

```sql
SELECT
  careplan_intervention_code,
  careplan_intervention_code_system
FROM careplan_intervention
WHERE careplan_intervention_patient_id = 18439
  AND careplan_intervention_encounter_id = 13875;
```

✔ 711028002 – Counseling
✔ 185796008 – Stop smoking monitoring

Intervention present.

---

# 6️⃣ API Verification

## getTobaccoScreeningData

Confirmed tobacco value saved via:

```
MacraFlowsheet/saveTobaccoPatientData
```

Returns:

```
"data": true
```

---

# 7️⃣ eCQM Specification Validation

Checked:

```sql
SELECT valueset_name, valueset_oid, qdm_category
FROM ecqm_specifications_2026
WHERE valueset_oid IN (
'2.16.840.1.113883.3.526.3.1170',
'2.16.840.1.113883.3.526.3.1189',
'2.16.840.1.113883.3.526.3.509'
);
```

### Findings:

| ValueSet                         | QDM Category |
| -------------------------------- | ------------ |
| Tobacco User                     | Attribute    |
| Tobacco Non User                 | Attribute    |
| Tobacco Use Cessation Counseling | Intervention |

⚠ Important discovery:

**Tobacco User / Non-User is QDM Category = Attribute**

---

# 8️⃣ Root Cause Identified

Original request JSON sent to CQM engine contained:

```json
"riskAssessmentList": [],
"tobaccoStatusList": []
```

### Problem:

* Smoking status saved only as **Assessment**
* Measure logic expects **QDM Attribute**
* `tobaccoStatusList` was NOT populated
* Therefore measure rule never triggered

So:

✔ Data saved in DB
❌ Not passed correctly to CQM engine

---

# 9️⃣ Working JSON Structure

Corrected JSON included:

### 1️⃣ riskAssessmentList (Screening performed)

```json
{
  "code": "72166-2",
  "codeSystemOID": "2.16.840.1.113883.6.1",
  "resultCode": "449868002"
}
```

### 2️⃣ tobaccoStatusList (Attribute)

```json
{
  "code": "449868002",
  "codeSystemOID": "2.16.840.1.113883.6.96",
  "startDate": 1769797800000
}
```

After adding:

✔ IPP = 1
✔ DEN = 1
✔ NUM = 1

Measure met.

---

# 🔟 Conclusion

## Issue Type

Mapping / QDM Category mismatch

## Not a DB Issue

* Data stored correctly
* Clinical element correct
* Intervention correct

## Actual Issue

Smoking status not framed as **QDM Attribute** in request payload

---

# 1️⃣1️⃣ Permanent Fix Required

When saving smoking status:

Application must:

1. Save to clinical elements (current behavior ✔)
2. Map to:

   * riskAssessmentList (Screening)
   * tobaccoStatusList (Attribute)
3. Ensure Attribute list is populated in CQM request

---

# 1️⃣2️⃣ Lessons Learned

* Always verify **QDM Category** in ecqm_specifications table
* DB presence ≠ Measure engine recognition
* CQM engine strictly depends on:

  * Correct QDM type
  * Correct list population
  * Correct codeSystemOID

---

# 1️⃣3️⃣ Quick Debug Checklist (Future)

If Measure 226 shows NOT MET:

✔ Check age
✔ Check qualifying visit
✔ Check patient_clinical_elements
✔ Check careplan_intervention
✔ Check ecqm_specifications qdm_category
✔ Verify tobaccoStatusList populated in request
✔ Validate resultCode + codeSystemOID

---

# 1️⃣4️⃣ Final Root Cause Statement (Short Version)

Smoking status was saved as Assessment but not passed as QDM Attribute (tobaccoStatusList) to the CQM engine. Since CMS138 expects Attribute category, measure evaluation failed.

---
