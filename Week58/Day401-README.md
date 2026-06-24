# MIPS Automation – CPT Code Removal & BMI Configuration Review (Dr. Atluri)

## Overview

This document captures the investigation, code changes, root cause analysis (RCA), validation activities, and deployment considerations related to:

1. **Removal of specific CPT II codes from automated claims generation**
2. **BMI Screening & Follow-Up (MIPS Measure #128) review for PY2026**
3. **Investigation of customer-reported missing BMI CPT codes**

This document is intended to serve as:

* Technical Knowledge Base
* Root Cause Analysis (RCA)
* Support Troubleshooting Guide
* Developer Reference
* Future Maintenance Documentation

---

# Background / Context

## Customer Request

Customer: **Netrin Health (Niharika Sarraf)**

Reported Issues:

### Issue 1 — CPT Code Removal

Customer requested that the following CPT II codes should **not be generated and pushed through claims processing**.

| CPT Code | Description                                          |
| -------- | ---------------------------------------------------- |
| G8421    | BMI not documented; no reason given                  |
| G8419    | BMI outside normal range and no follow-up documented |
| G8428    | Medication list not documented; no reason given      |
| 1123F-8P | ACP not documented; reason not specified             |

---

### Issue 2 — BMI Screening & Follow-Up Review

Customer stated:

> BMI measure was active in 2024, inactive in 2025, and reactivated in 2026.

Customer provided sample patients where:

> BMI codes were not generated despite appropriate documentation being completed.

Sample Patients:

| Account # | DOS        |
| --------- | ---------- |
| 011297    | 02/03/2026 |
| 011297    | 05/05/2026 |
| 5418      | 03/11/2026 |

---

# Problem Statement

Customer expected BMI CPT II codes to be automatically generated for the above encounters.

Observed behavior:

* Follow-up documentation existed.
* No BMI CPT code generated.
* Customer assumed BMI automation was failing.

Required investigation:

1. Verify Measure 128 configuration.
2. Verify encounter eligibility.
3. Verify BMI documentation.
4. Verify follow-up documentation.
5. Verify automation logic.
6. Determine why CPT was not generated.

---

# Impact

## Business Impact

Potential concerns:

* Missing MIPS quality reporting CPT codes.
* Reduced quality measure performance.
* Customer confusion regarding BMI measure requirements.

---

## Technical Impact

Automation behaves correctly but customer documentation was incomplete.

Missing BMI values prevent CPT generation.

---

# Environment Details

## Reporting Year

2026

---

## Measure

MIPS #128

### BMI Screening and Follow-Up

| BMI Range   | Follow-Up Required | CPT    |
| ----------- | ------------------ | ------ |
| 18.5 – 24.9 | No                 | G8420  |
| ≥ 25        | Yes                | G8417  |
| < 18.5      | Yes                | G8418  |
| ≥ 25        | No                 | No CPT |
| < 18.5      | No                 | No CPT |

---

## BMI Logic

```text
Qualifying Encounter
        ↓
Age >= 18
        ↓
Not Hospice/Palliative/Pregnancy
        ↓
BMI Documented
        ↓

BMI Normal      → G8420

BMI High        → Follow-up Required → G8417

BMI Low         → Follow-up Required → G8418

No BMI          → No CPT
```

---

# System Components Involved

## Database Tables

### Configuration

```sql
hedis_configuration
hedis_measure_provider_configuration
```

### Patient Data

```sql
patient_registration
chart
encounter
patient_clinical_elements
problem_list
patient_assessments
```

### Billing

```sql
service_detail
cpt
```

---

## Services

### HEDIS Automation

```java
saveServicesBasedOnHedisMeasuresConfig()
```

### Vitals Extraction

```java
getVitals()
```

### BMI CPT Determination

```java
getBMICode()
```

### Existing Service Lookup

```java
getExistingServiceMap()
```

---

# Investigation Timeline

## Step 1 – Verify HEDIS Configuration

### Query

```sql
SELECT *
FROM hedis_configuration
WHERE hedis_configuration_reporting_year = 2026;
```

### Result

```text
2026 Active = TRUE
```

---

## Step 2 – Verify Measure 128 Configuration

### Query

```sql
SELECT *
FROM hedis_measure_provider_configuration
WHERE hedis_measure_provider_configuration_reporting_year = 2026
AND hedis_measure_provider_configuration_measure_id LIKE '%128%';
```

### Result

Configured for:

```text
Provider 1
Provider 11
Provider 12
```

Measure 128 is correctly enabled.

---

## Step 3 – Verify Sample Encounters

### Patient 11297

DOS:

```text
02/03/2026
05/05/2026
```

### Patient 5518

DOS:

```text
03/11/2026
```

Encounter IDs identified successfully.

---

## Step 4 – Check Generated BMI CPT Codes

### Query

```sql
SELECT sd.service_detail_id,
       cp.cpt_cptcode
FROM service_detail sd
JOIN cpt cp
ON cp.cpt_id = sd.service_detail_cptid
WHERE sd.service_detail_patientid = 11297
AND cp.cpt_cptcode IN
(
'G8420',
'G8417',
'G8418',
'G2181',
'G8419'
);
```

### Result

```text
0 rows
```

No BMI CPT generated.

---

## Step 5 – Verify Follow-Up Documentation

Relevant GWIDs:

| Purpose               | GWID                |
| --------------------- | ------------------- |
| BMI Value             | 0000200200100025000 |
| Underweight Follow-Up | 0000423900000000003 |
| Overweight Follow-Up  | 0530425900000000001 |

---

### Overweight Follow-Up Query

```sql
SELECT *
FROM patient_clinical_elements
WHERE patient_clinical_elements_gwid='0530425900000000001';
```

### Result

Found for:

```text
11297 – 02/03/2026
11297 – 05/05/2026
5518  – 03/11/2026
```

Follow-up documentation exists.

---

## Step 6 – Verify BMI Documentation

### Query

```sql
SELECT *
FROM patient_clinical_elements
WHERE patient_clinical_elements_gwid='0000200200100025000'
AND patient_clinical_elements_created_on
BETWEEN '2026-01-01' AND '2026-12-31';
```

### Result

```text
0 rows
```

No BMI value documented in 2026.

---

## Step 7 – Inspect Clinical Elements

Encounter inspection showed:

```text
Height = Present

Overweight Follow-Up = Present

BMI = Missing
```

Example:

```text
Encounter 1017818

Height = 175.2600
Overweight = 1

BMI Value = Missing
```

---

# Root Cause Analysis

## Root Cause

BMI CPT generation requires:

1. BMI value documented.
2. Follow-up documented (for abnormal BMI).

Only follow-up documentation was present.

BMI value itself was missing.

Therefore:

```java
if ("BMI".equalsIgnoreCase(displayName))
{
    bmiCode = getBMICode(curValue, followUpDocumented);
}
```

never generated a CPT because BMI data was unavailable.

---

## Why CPT Was Not Generated

### Patient 11297

| DOS        | BMI     | Follow-Up |
| ---------- | ------- | --------- |
| 02/03/2026 | Missing | Present   |
| 05/05/2026 | Missing | Present   |

---

### Patient 5518

| DOS        | BMI     | Follow-Up |
| ---------- | ------- | --------- |
| 03/11/2026 | Missing | Present   |

---

### Result

```text
BMI Missing
+
Follow-Up Present
=
No CPT Generated
```

Expected behavior.

---

# Detailed Technical Findings

## BMI CPT Generation Logic

### Current Logic

```java
private String getBMICode(String curValue, boolean followUp) {
    try {
        if (curValue != null && !curValue.trim().isEmpty()) {

            double bmi = Double.parseDouble(curValue.trim());

            if (bmi >= 18.5 && bmi <= 24.9)
                return "G8420";

            return followUp
                ? (bmi > 24.9 ? "G8417" : "G8418")
                : "";
        }
    } catch (NumberFormatException e) {
        GlaceLogger.LogException(e);
    }

    return "";
}
```

---

### Behavior

| Scenario                | CPT   |
| ----------------------- | ----- |
| Normal BMI              | G8420 |
| High BMI + Follow-Up    | G8417 |
| Low BMI + Follow-Up     | G8418 |
| High BMI + No Follow-Up | Blank |
| Low BMI + No Follow-Up  | Blank |
| BMI Missing             | Blank |

---

# Code Changes

## Requirement

Remove generation of:

```text
G8421
G8419
G8428
1123F-8P
```

---

## BMI Change

### Before

```java
return followUp
       ? (bmi > 24.9 ? "G8417" : "G8418")
       : "G8419";
```

### After

```java
return followUp
       ? (bmi > 24.9 ? "G8417" : "G8418")
       : "";
```

### Effect

G8419 can no longer be generated.

---

## Medication Review Change

### Before

```java
currentMedicationCodesList =
Arrays.asList("G8427", "G8428");
```

### After

```java
currentMedicationCodesList =
Arrays.asList("G8427");
```

### Effect

G8428 removed.

---

## Existing Service Map Change

### Before

```java
bmiCodesList =
Arrays.asList(
"G8420",
"G8417",
"G8418",
"G2181",
"G8419"
);
```

### After

```java
bmiCodesList =
Arrays.asList(
"G8420",
"G8417",
"G8418",
"G2181"
);
```

### Effect

G8419 removed from BMI automation processing.

---

## Commit

```text
Commit ID: 58021
```

Purpose:

```text
Remove unsupported BMI/Medication/ACP negative CPT codes
from automatic claims generation.
```

---

# SQL Analysis and Scripts

## Investigation Query – Verify HEDIS Year

```sql
SELECT *
FROM hedis_configuration
WHERE hedis_configuration_reporting_year = 2026;
```

Purpose:

* Verify HEDIS PY2026 enabled.

Type:

```text
Validation Query
```

---

## Investigation Query – Measure Configuration

```sql
SELECT *
FROM hedis_measure_provider_configuration
WHERE hedis_measure_provider_configuration_reporting_year = 2026
AND hedis_measure_provider_configuration_measure_id LIKE '%128%';
```

Purpose:

* Verify Measure 128 assignment.

Type:

```text
Validation Query
```

---

## Investigation Query – BMI Documentation

```sql
SELECT *
FROM patient_clinical_elements
WHERE patient_clinical_elements_gwid='0000200200100025000'
AND patient_clinical_elements_created_on
BETWEEN '2026-01-01'
AND '2026-12-31';
```

Purpose:

* Verify BMI values documented.

Type:

```text
Investigation Query
```

---

## Investigation Query – Follow-Up Documentation

```sql
SELECT *
FROM patient_clinical_elements
WHERE patient_clinical_elements_gwid='0530425900000000001';
```

Purpose:

* Verify overweight follow-up.

Type:

```text
Investigation Query
```

---

# Validation and Testing

## Configuration Validation

Verified:

```text
2026 Active = TRUE
Measure 128 Configured = TRUE
```

---

## Patient Validation

Verified:

```text
Follow-up documentation exists.
BMI values missing.
```

---

## Billing Validation

Verified:

```text
No BMI CPT created.
```

Expected behavior.

---

# Fixes and Workarounds

## Permanent Fix

Removed generation of:

```text
G8421
G8419
G8428
1123F-8P
```

Commit:

```text
58021
```

---

## Workaround

Users must document:

```text
BMI Value
+
Follow-Up
```

for abnormal BMI encounters.

Follow-up alone is insufficient.

---

# Risks and Side Effects

## Low Risk

Changes only remove unwanted CPT generation.

---

## Potential Impact

Patients previously receiving:

```text
G8419
G8428
```

will no longer receive those CPT codes.

This is expected per customer request.

---

# Pending Work

## Deployment

Commit:

```text
58021
```

Awaiting deployment to customer environment.

---

## Customer Notification

Inform customer once deployment is completed.

---

# Lessons Learned

## Key Findings

### Follow-Up ≠ BMI Documentation

Presence of:

```text
Overweight
Underweight
```

does not imply BMI value exists.

---

### Verify Source Data First

Always validate:

```text
patient_clinical_elements
```

before investigating automation code.

---

### Recommended Debugging Sequence

1. Verify HEDIS configuration.
2. Verify provider measure assignment.
3. Verify encounter eligibility.
4. Verify BMI GWID data.
5. Verify follow-up GWID data.
6. Verify generated CPT.
7. Review automation logic.

---

# Final Conclusion

### Issue 1 – CPT Removal

Resolved.

The requested CPT codes:

```text
G8421
G8419
G8428
1123F-8P
```

have been removed from automatic CPT generation logic (Commit ID: 58021).

---

### Issue 2 – Missing BMI CPT Codes

Not a system defect.

Investigation confirmed:

```text
Follow-up documentation = Present
BMI documentation = Missing
```

for all customer-provided sample encounters.

Since BMI values were not documented, BMI CPT codes were correctly not generated by the automation logic.
