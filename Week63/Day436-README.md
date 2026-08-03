# README

# CPT II Codes Not Populating in Claims Due to Unsigned Clinical Notes

---

# Overview

This document describes the investigation performed for the customer-reported issue where **CPT II codes were not appearing in claims** for **Dr. Asif Farooqui's** patients despite clinical documentation being completed.

The investigation concluded that the affected encounters **had not been signed**, preventing CPT II service generation according to the configured workflow.

---

# Issue Summary

### Reported By

Paddy G (Technical Support)

### Customer

* Dr. Asif Farooqui
* Tracy Behrens

### Reported Problem

Customer reported that:

> CPT II codes were not appearing in claims even though the required documentation had been entered into the EHR.

Examples from June and July were provided for validation.

---

# Background

Earlier communication regarding this feature confirmed the following workflow.

## Auto Populate Button

Users can manually generate CPT II codes before signing by clicking:

```
Superbill
    ↓
Auto Populate
```

## Automatic Generation

If Auto Populate is not used,

CPT II services are automatically generated when the clinical note is signed.

Exception:

* HbA1c follows separate logic.

---

# Investigation

The requested task was:

> Fetch the list of patients whose claims should contain CPT II codes based on their EHR documentation.

The investigation focused on verifying:

* Clinical documentation
* Encounter information
* Note signing status
* CPT II generation

---

# Database Verification

## Table Verified

```
leaf_patient
```

Important columns reviewed:

```
leaf_patient_sign_userid
leaf_patient_completed_on
leaf_patient_encounter_id
leaf_patient_patient_id
```

These fields determine whether the clinical note has been signed.

---

# SQL Used

To verify signed encounters:

```sql
SELECT
    lp.leaf_patient_id,
    lp.leaf_patient_patient_id,
    pr.patient_registration_accountno,
    CONCAT_WS(' ',
        pr.patient_registration_first_name,
        pr.patient_registration_mid_initial,
        pr.patient_registration_last_name
    ) AS patient_name,
    lp.leaf_patient_encounter_id,
    lp.leaf_patient_sign_userid,
    ep.emp_profile_fullname AS signed_by,
    lp.leaf_patient_completed_on
FROM leaf_patient lp
JOIN patient_registration pr
    ON pr.patient_registration_id = lp.leaf_patient_patient_id
LEFT JOIN emp_profile ep
    ON ep.emp_profile_empid = lp.leaf_patient_sign_userid
WHERE lp.leaf_patient_sign_userid IS NOT NULL
ORDER BY lp.leaf_patient_completed_on DESC;
```

---

# Findings

Investigation showed that:

* Clinical documentation existed.
* Required EHR documentation had been entered.
* The affected encounters had **not been signed**.
* Because the notes remained unsigned, CPT II services were **not generated**.
* Therefore, CPT II codes did not appear in claims.

---

# Root Cause Analysis (RCA)

## Root Cause

Clinical notes were **not signed**.

Since the configured workflow generates CPT II services upon note signing, the CPT II codes were never created.

---

# Workflow

```
Clinical Documentation
        │
        ▼
Save Clinical Data
        │
        ▼
Note Signed?
        │
   ┌────┴────┐
   │         │
  YES        NO
   │         │
   ▼         ▼
Generate     No CPT II
CPT II       Generation
Codes
   │
   ▼
Insert Service
Detail
   │
   ▼
Claims
```

---

# Impact

Affected patients:

* Clinical documentation present
* CPT II codes missing
* Claims did not contain CPT II services

Reason:

```
Unsigned Notes
```

---

# Resolution

Requested action:

> Ask Dr. Asif Farooqui to sign the clinical notes.

Once signed:

* CPT II services will be generated.
* CPT II codes will populate in claims according to the configured workflow.

---

# Customer Communication

The following response was sent.

> Dear Paddy,
>
> We verified the reported encounters and found that the clinical notes have not been signed.
>
> Please ask Dr. Farooqui to sign the notes. Once the notes are signed, the CPT II codes will populate in the claims.
>
> Kindly let us know if you have any further concerns.
>
> Thanks,
> Gobala Krishnan

---

# Conclusion

The issue was **not caused by missing configuration or code defects**.

The CPT II generation logic behaved as designed.

The missing CPT II codes were due to **unsigned clinical notes**, preventing automatic CPT II generation and subsequent inclusion in claims.

---

# Lessons Learned

* Always verify note signing status before investigating CPT II generation issues.
* The `leaf_patient` table is the primary source for determining whether a note has been signed.
* If documentation exists but CPT II codes are missing, first confirm:

  * Clinical note is signed.
  * `leaf_patient_sign_userid` is populated.
  * `leaf_patient_completed_on` has a valid timestamp.
* Unsigned encounters will not generate CPT II services under the current workflow, resulting in no CPT II codes appearing in claims.
