
# README - CMS134 Depression Screening and Follow-Up Plan (QRDA-I) Investigation, Debugging, and Root Cause Analysis

## Overview

This document captures the complete investigation performed for the **CMS134 - Preventive Care and Screening: Screening for Depression and Follow-Up Plan** measure, specifically related to:

* QRDA-I generation
* Depression Screening Positive/Negative result mapping
* Follow-Up Intervention population
* Innovaccer QRDA consumption issues
* PCAWH (Premier Care Associates of West Hudson) and Sai Medical Center validation
* EMeasureUtils intervention processing logic
* Clinical Elements ↔ QDM ↔ QRDA transformation flow

This document serves as:

* Technical Knowledge Base
* Root Cause Analysis (RCA)
* Developer Reference
* Support Troubleshooting Guide
* Future Maintenance Documentation

---

# Background / Context

## Measure

CMS134 - Preventive Care and Screening: Screening for Depression and Follow-Up Plan

### Positive Depression Screening

Requires:

* Depression Screening
* Positive Result
* Appropriate Follow-Up Plan

### Negative Depression Screening

Requires:

* Depression Screening
* Negative Result

---

## Business Requirement

Innovaccer requested clarification regarding:

1. Which QRDA nodes represent:

   * Positive Depression Screening
   * Negative Depression Screening

2. How Follow-Up Plan is documented

3. Why Depression scores/results were not being interpreted correctly from QRDA-I files

---

# Problem Statement

## Reported Issue

Practices including:

* Premier Care Associates of West Hudson (PCAWH)
* Sai Medical Center

reported:

* Depression screening scores not appearing in Innovaccer
* Follow-Up Plan not being recognized
* QRDA-I seemingly not populating expected data

---

## Initial Symptoms

For Patient:

```text
Patient ID = 377165
Provider ID = 1361
Measure = 134
```

Database showed:

```text
Positive PHQ-9 Screening
Referral documented
Follow-Up documented
Numerator = 1
```

Yet QRDA validation raised concerns regarding:

```text
Follow-Up intervention population
Screening score visibility
```

---

# Impact

## Business Impact

### If unresolved

* CMS134 numerator may fail
* Depression Follow-Up may not be recognized
* Innovaccer quality scores may be incorrect
* Reporting discrepancies between Glace and Innovaccer

### Affected Areas

* QRDA-I exports
* eCQM processing
* MIPS reporting
* Innovaccer ingestion

---

# Environment Details

## Database

```text
PostgreSQL
```

Database:

```text
ciama_nextrelease
```

---

## Application

```text
Glace EMR Backend
```

Path:

```text
/home/software/git/glaceemr_backend_new/
```

---

## Key File

```java
/home/software/git/glaceemr_backend_new/src/main/java/com/glenwood/glaceemr/server/application/Bean/macra/ecqm/EMeasureUtils.java
```

Method:

```java
getInterventionFromCNM()
```

---

# System Components Involved

| Component                        | Purpose                         |
| -------------------------------- | ------------------------------- |
| clinical_elements                | Clinical element master         |
| clinical_elements_options        | Clinical element option values  |
| patient_clinical_elements        | Patient documented plan items   |
| cnm_code_system                  | SNOMED mappings                 |
| risk_assessment                  | Depression screening            |
| quality_measures_patient_entries | Measure calculation result      |
| EMeasureUtils                    | Builds Intervention QDM objects |
| ExportQDM                        | QDM generation                  |
| QRDA Generator                   | QRDA-I XML generation           |
| Innovaccer                       | Downstream consumer             |

---

# Investigation Timeline

## Step 1 - Verify Measure JSON

Location:

```bash
/mnt/vs22shared/eswar/tmp/mu/ECQM/2026
```

Files:

```text
134.json
112.json
113.json
236.json
...
```

Observation:

```text
134.json contains Follow-Up value set
SNOMED 88848003 present
```

No issue found in measure JSON.

---

## Step 2 - Verify Risk Assessment

Query:

```sql
SELECT
    ra.risk_assessment_id,
    ra.risk_assessment_patient_id,
    ra.risk_assessment_screening_name,
    ra.risk_assessment_result_description,
    ra.risk_assessment_result_code
FROM risk_assessment ra
WHERE ra.risk_assessment_patient_id = 377165
AND ra.risk_assessment_code IN ('73831-0','73832-8');
```

Result:

```text
PHQ-9 Adult
Positive
LOINC = 73832-8
```

Finding:

```text
Depression screening exists.
```

---

## Step 3 - Verify Clinical Element Documentation

Query returned:

```text
Referral for Depression Adult
SNOMED = 61801003

Follow-up for depression - adult
SNOMED = 88848003
```

Finding:

```text
Both Referral and Follow-Up documented correctly.
```

---

## Step 4 - Verify Clinical Element Configuration

### Referral

```text
GWID = 0000409300000000061
SNOMED = 61801003
```

### Follow-Up

```text
GWID = 0000409300000000065
SNOMED = 88848003
```

Configuration exists.

---

## Step 5 - Verify CNM Mapping

Query showed:

```text
Referral:
CNM Code = 61801003

Follow-Up:
CNM Code = 91310009
Clinical Element SNOMED = 88848003
```

Important finding:

Follow-Up uses:

```text
clinical_elements_snomed = 88848003
```

but CNM result code becomes:

```text
91310009
```

---

## Step 6 - Debug ExportQDM

Logs:

```text
FOLLOW UP RAW HIBERNATE RESULT

CODE          = 88848003
RESULT CODE   = 91310009
GWID          = 0000409300000000065
STATUS        = 0
```

Finding:

Follow-Up record successfully reaches QDM layer.

---

## Step 7 - Debug EMeasureUtils

Log:

```text
CNM INTERVENTION INPUT

GWID = 0000409300000000065
CODE = 88848003
STATUS = 0
RESULT CODE = 91310009
```

Generated:

```text
FINAL INTERVENTION

CODE = 88848003
STATUS = 0
```

Critical observation:

```text
Status remains 0.
```

---

## Step 8 - Generated QRDA Review

Generated QRDA contained:

### Assessment

```xml
<code code="73832-8"/>
```

### Positive Result

```xml
<value code="428181000124104"/>
```

But:

```text
No Follow-Up Intervention node generated
```

Finding:

Follow-Up documented in database
Follow-Up present in QDM
Follow-Up missing from QRDA

---

# Root Cause Analysis

## Primary Root Cause

Follow-Up interventions enter QDM with:

```text
STATUS = 0
```

However CMS134 logic expects:

```text
Intervention, Performed
```

which requires:

```text
STATUS = 2 (Completed/Performed)
```

or equivalent mapped status.

---

## Why Referral Appears

Referral records contain:

```text
STATUS = 2
```

Thus they satisfy:

```cql
Intervention, Order
```

and are included.

---

## Why Follow-Up Does Not Appear

Follow-Up records:

```text
STATUS = 0
```

Generated intervention:

```text
CODE=88848003
STATUS=0
```

The QRDA generator ignores them because:

```text
Not recognized as Performed Intervention
```

---

## Secondary Issue

Current code only updates status when:

```java
codeDescription != null
```

Follow-Up record:

```text
DESCRIPTION = null
```

Therefore:

```java
else {
    interventionObj.setStatus(eachObj.getStatus());
}
```

executes.

Result:

```text
STATUS remains 0
```

---

# Detailed Technical Findings

## Existing Code

```java
if(eachObj.getCodeDescription()!=null &&
   !eachObj.getCodeDescription().equals(""))
{
    ...
}
else
{
    interventionObj.setStatus(eachObj.getStatus());
}
```

---

## Althaf Proposed Change

```java
else {
    interventionObj.setStatus(
        eachObj.getStatus() != 0
        ? eachObj.getStatus()
        : 2
    );
}
```

Purpose:

```text
Convert empty/zero status
to Performed status.
```

---

## Problem

The proposed fix was placed inside:

```java
if(codeDescription != null)
```

block.

However Follow-Up records have:

```text
codeDescription = null
```

Therefore:

```text
Fix block never executes.
```

---

## Actual Behavior

Current execution:

```java
DESCRIPTION = null

-> enters else block

interventionObj.setStatus(0)
```

Result:

```text
Follow-Up omitted from QRDA.
```

---

# SQL Analysis and Scripts

## Investigation Queries

### Verify Depression Screening

```sql
SELECT
    ra.risk_assessment_id,
    ra.risk_assessment_patient_id,
    ra.risk_assessment_screening_name,
    ra.risk_assessment_result_description,
    ra.risk_assessment_result_value
FROM risk_assessment ra
WHERE ra.risk_assessment_patient_id = 377165
AND ra.risk_assessment_code IN ('73831-0','73832-8');
```

Purpose:

```text
Verify Positive/Negative screening.
```

Type:

```text
Validation Query
```

---

### Verify Follow-Up Clinical Elements

```sql
SELECT
    ce.clinical_elements_name,
    ce.clinical_elements_snomed
FROM clinical_elements ce
WHERE clinical_elements_name ILIKE '%depression%'
AND (
      clinical_elements_name ILIKE '%referral%'
   OR clinical_elements_name ILIKE '%follow-up%'
);
```

Purpose:

```text
Verify Depression Follow-Up definitions.
```

Type:

```text
Investigation Query
```

---

### Verify Patient Documentation

```sql
SELECT
    pce.patient_clinical_elements_id,
    ce.clinical_elements_name,
    ce.clinical_elements_snomed
FROM patient_clinical_elements pce
JOIN clinical_elements ce
ON ce.clinical_elements_gwid =
   pce.patient_clinical_elements_gwid;
```

Purpose:

```text
Verify actual patient documentation.
```

Type:

```text
Validation Query
```

---

### Verify Measure Calculation

```sql
SELECT
    quality_measures_patient_entries_patient_id,
    quality_measures_patient_entries_numerator
FROM quality_measures_patient_entries
WHERE quality_measures_patient_entries_measure_id='134';
```

Purpose:

```text
Verify numerator calculation.
```

Result:

```text
Numerator = 1
```

---

# Code Changes

## File

```java
EMeasureUtils.java
```

Method:

```java
getInterventionFromCNM()
```

---

## Before

```java
else
{
    interventionObj.setStatus(eachObj.getStatus());
}
```

---

## Proposed

```java
else
{
    interventionObj.setStatus(
        eachObj.getStatus() != 0
        ? eachObj.getStatus()
        : 2
    );
}
```

---

## Recommended Fix

Apply defaulting regardless of description.

Example:

```java
if(eachObj.getStatus()==0 &&
   ("88848003".equals(eachObj.getCode())
    || "61801003".equals(eachObj.getCode())))
{
    interventionObj.setStatus(2);
}
```

or

```java
interventionObj.setStatus(
    eachObj.getStatus() != 0
    ? eachObj.getStatus()
    : 2
);
```

after description logic.

---

# Validation and Testing

## Database Validation

Confirmed:

```text
Positive PHQ-9
Referral documented
Follow-Up documented
```

---

## QDM Validation

Confirmed:

```text
CODE = 88848003
```

reaches:

```text
ClinicalDataQDM
```

---

## Intervention Validation

Confirmed:

```text
Intervention created
```

but:

```text
STATUS = 0
```

---

## QRDA Validation

Confirmed:

### Present

```text
Encounter
Assessment
Positive Result
Referral
```

### Missing

```text
Follow-Up Intervention
```

---

# Innovaccer Reference Mapping

## Negative Screening

LOINC:

```text
73832-8
```

SNOMED:

```text
428171000124102
```

Reference file: 

---

## Positive Screening

LOINC:

```text
73832-8
```

SNOMED:

```text
428181000124104
```

Reference file: 

---

## Follow-Up

SNOMED:

```text
88848003
```

Reference file: 

---

# Workarounds

## Temporary

Use Referral documentation:

```text
61801003
```

because it already generates:

```text
STATUS = 2
```

and appears in QRDA.

---

# Deployment Considerations

Before deployment:

1. Test Positive Screening
2. Test Negative Screening
3. Test Referral Only
4. Test Follow-Up Only
5. Validate QRDA XML
6. Validate Innovaccer ingestion

---

# Risks and Side Effects

## Risk

Changing:

```java
status=0 -> status=2
```

globally may affect:

* Other interventions
* Other measures
* Historical records

---

## Recommendation

Restrict status conversion to:

```text
Depression Follow-Up GWIDs
```

or

```text
Depression Follow-Up SNOMED codes
```

only.

---

# Pending Items

## Open Questions

1. Why Follow-Up documentation stores:

```text
STATUS = 0
```

instead of:

```text
STATUS = 2
```

2. Is this issue limited to:

```text
GWID 0000409300000000065
GWID 0000409300000000063
```

or all Follow-Up interventions?

3. Does QRDA generator explicitly filter:

```text
STATUS != 2
```

records?

---

# Lessons Learned

## Debugging Insights

Always validate:

```text
Database
→ ClinicalDataQDM
→ Intervention
→ Request Object
→ QRDA XML
```

instead of stopping at numerator calculation.

---

## Logging Improvements

Add logs:

```java
GWID
SNOMED
STATUS
DESCRIPTION
```

before QRDA generation.

---

## Monitoring Recommendations

Create validation checks for:

```text
Positive Screening + Follow-Up documented
but Follow-Up missing from QRDA
```

---

## Final Conclusion

The investigation confirmed that:

* Depression screening data is documented correctly.
* Follow-Up clinical elements are configured correctly.
* Follow-Up reaches the QDM layer successfully.
* The Follow-Up intervention is created in `getInterventionFromCNM()`.
* The intervention is assigned `STATUS = 0`.
* QRDA generation does not treat `STATUS = 0` as a valid performed intervention.
* As a result, Follow-Up SNOMED `88848003` is omitted from the generated QRDA-I XML.

The most likely permanent fix is to ensure Depression Follow-Up interventions are transformed into a valid performed status (`STATUS = 2`) before QRDA generation while limiting the change to depression-specific follow-up records to avoid side effects in other measures.
