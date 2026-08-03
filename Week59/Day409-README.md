
# README - MIPS Performance Report "Not Met" Patient List Fails with `JSON.parse: unexpected end of data` (Measure 134)

---

# Overview

This document captures the complete investigation, analysis, troubleshooting process, and resolution for an issue encountered while opening the **"Not Met" patient list** for **MIPS Measure 134** in PCAWH.

The UI displayed the following browser error:

```
SyntaxError: JSON.parse: unexpected end of data
```

Although the browser reported a JSON parsing error, the investigation showed that the problem originated from backend processing of patient demographic lookup data.

This document serves as:

- Root Cause Analysis (RCA)
- Technical Documentation
- Troubleshooting Guide
- Developer Reference
- Support Knowledge Base

---

# Background / Context

The MIPS Performance Report page allows users to view patients belonging to a selected measure.

Example request:

```
MIPSPerformanceReport.Action
```

Request Body

```
isNotMet=true
provider=-1
searchMode=2
mode=2
criteria=1
measureId=134
patientList=2739
ssn=204503813
year=2026
filterExcludeHospitalVisit=false
filterExcludeERVisit=false
filterExcludeNursingHomeVisit=false
filterExcludeAssistedLivingVisit=false
GlaceAjaxRequest=true
```

The selected patient:

| Field | Value |
|---------|------|
| Patient ID | 2739 |
| Measure | 134 |
| Reporting Year | 2026 |
| Provider ID | 41 |
| Provider | Mariza De Brito M.D. |
| Provider TIN | 204503813 |

---

# Problem Statement

When opening the **Not Met patient list** for Measure 134, the UI failed with:

```
SyntaxError:
JSON.parse: unexpected end of data
```

Firefox Network tab:

```
Status : 200 OK
```

However,

```
Response Body
(empty)
```

The browser attempted

```javascript
JSON.parse(response)
```

which failed because the backend returned an empty payload.

---

# Impact

## User Impact

Users could not open the "Not Met" patient details for Measure 134.

---

## Technical Impact

The backend swallowed an exception and returned an empty response.

The frontend expected valid JSON.

Result:

```
JSON.parse(...)
```

failed.

---

# Environment Details

Environment:

```
PCAWH
```

Reporting Year

```
2026
```

Measure

```
134
```

Patient

```
2739
```

Provider

```
41
```

TIN

```
204503813
```

---

# System Components Involved

## Legacy Application

```
MIPSPerformanceReport.Action
```

Class

```
com.glenwood.hcare.actions.Measures.MIPSPerformanceReportAction
```

---

## Spring Backend

Controller

```
QPPPerformanceController
```

Endpoint

```
POST
/QPPPerformance/getPatient
```

---

## Service

```
MeasureCalcServiceImpl.getPatient(...)
```

---

## Helper Methods

```
getPatientGranularRace()

getPatientGranularethnicity()
```

---

## Database Tables

```
quality_measures_patient_entries
```

```
patient_registration
```

```
patient_ins_detail
```

```
billinglookup
```

```
emp_profile
```

---

# Request Flow

```
Browser

↓

MIPSPerformanceReport.Action

↓

CustomURLConnection

↓

QPPPerformance/getPatient

↓

MeasureCalcServiceImpl.getPatient()

↓

MIPSPatientInformation

↓

JSON Response

↓

Browser JSON.parse()
```

---

# Investigation Timeline

## Step 1

Verified browser request.

```
Status : 200 OK
```

Browser failed while parsing JSON.

Initial suspicion:

Backend returned empty response.

---

## Step 2

Reviewed

```
MIPSPerformanceReportAction.java
```

Mode

```
mode = 2
```

Request forwarded to

```
QPPPerformance/getPatient
```

---

## Step 3

Reviewed

```
QPPPerformanceController.java
```

Controller simply delegates to

```
measureService.getPatient(...)
```

---

## Step 4

Reviewed

```
MeasureCalcServiceImpl.getPatient(...)
```

Investigated:

- Native SQL
- joins
- criteria
- provider
- TIN
- patient filtering
- Not Met logic

---

## Step 5

Validated database records.

Verified:

- quality measure entry exists
- provider exists
- TIN matches
- patient exists
- insurance exists

Everything was correct.

---

## Step 6

Verified Not Met calculation.

Remaining count:

```
1
```

Therefore patient correctly belongs to Not Met list.

---

## Step 7

Investigated helper methods.

Discovered:

```
getPatientGranularRace()

getPatientGranularethnicity()
```

Both execute

```java
patientDetails.get(0)
```

without checking whether the query returned any rows.

---

## Step 8

Investigated billinglookup.

Found missing lookup values.

---

# Root Cause Analysis

## Root Cause

Patient demographic data contained a granular ethnicity code that did not match the reference data in the `billinglookup` table.

Patient record:

```
patient_registration_granular_ethnicity_code

,137-8
```

Application trims the comma:

```
137-8
```

Then executes

```java
SELECT
blook_name
FROM billinglookup
WHERE
blook_group=756
AND
blook_code_value='137-8'
```

Result:

```
0 rows
```

The helper method executed

```java
patientethnicityDetails.get(0);
```

which can throw an `IndexOutOfBoundsException`.

The controller catches the exception and returns an empty `EMRResponseBean`.

Legacy UI receives an empty response.

Browser attempts:

```javascript
JSON.parse(...)
```

Result:

```
SyntaxError:
unexpected end of data
```

---

# Detailed Technical Findings

## Controller

```
QPPPerformanceController.getPatient()
```

Calls

```
measureService.getPatient(...)
```

---

## Service

```
MeasureCalcServiceImpl
```

Builds patient list.

Creates

```
MIPSPatientInformation
```

Calls

```
getPatientGranularRace()

getPatientGranularethnicity()
```

---

## Risky Code

```java
patientRaceDetails.get(0);
```

```java
patientethnicityDetails.get(0);
```

No empty list validation.

---

# SQL Analysis and Scripts

## Investigation Queries

### Verify Measure Entry

```sql
SELECT
quality_measures_patient_entries_patient_id,
quality_measures_patient_entries_measure_id,
quality_measures_patient_entries_provider_id,
quality_measures_patient_entries_tin,
quality_measures_patient_entries_criteria,
quality_measures_patient_entries_denominator,
quality_measures_patient_entries_numerator
FROM quality_measures_patient_entries
WHERE
quality_measures_patient_entries_patient_id=2739
AND quality_measures_patient_entries_measure_id='134'
AND quality_measures_patient_entries_reporting_year=2026;
```

Purpose

- Validate patient measure entry.

---

### Verify Provider

```sql
SELECT
emp_profile_empid,
emp_profile_fullname,
emp_profile_ssn
FROM emp_profile
WHERE emp_profile_empid=41;
```

Purpose

Validate provider.

---

### Verify Patient Registration

```sql
SELECT
patient_registration_granular_race_code,
patient_registration_granular_ethnicity_code
FROM patient_registration
WHERE patient_registration_id=2739;
```

Result

```
Race

-1

Ethnicity

,137-8
```

---

### Verify Billing Lookup

```sql
SELECT *
FROM billinglookup
WHERE
blook_group=756
AND
blook_code_value='137-8';
```

Result

```
0 rows
```

---

### Verify Insurance

```sql
SELECT *
FROM patient_ins_detail
WHERE
patient_ins_detail_patientid=2739
AND patient_ins_detail_instype=1
AND patient_ins_detail_isactive=true;
```

---

### Verify Remaining Count

```sql
SELECT
quality_measures_patient_entries_denominator,
quality_measures_patient_entries_denominator_exclusion,
quality_measures_patient_entries_denominator_exception,
quality_measures_patient_entries_numerator,
quality_measures_patient_entries_numerator_exclusion
FROM quality_measures_patient_entries
WHERE
quality_measures_patient_entries_patient_id=2739
AND quality_measures_patient_entries_measure_id='134'
AND quality_measures_patient_entries_reporting_year=2026;
```

Confirmed patient belongs to Not Met.

---

## Workaround Applied

Patient demographic code was updated.

Before

```sql
SELECT patient_registration_granular_ethnicity_code
FROM patient_registration
WHERE patient_registration_id=2739;
```

Result

```
,137-8
```

Applied update

```sql
UPDATE patient_registration
SET patient_registration_granular_ethnicity_code=',2137-8'
WHERE patient_registration_id=2739;
```

Validation

```sql
SELECT patient_registration_granular_ethnicity_code
FROM patient_registration
WHERE patient_registration_id=2739;
```

Purpose:

Correct the patient's granular ethnicity code to match the expected reference value used by the application.

> **Note:** This update is a **data correction/workaround**. It should only be applied if `2137-8` is the correct code according to the organization's demographic coding standard. The underlying code should still be made resilient to missing lookup values.

---

# Code Improvement Recommendations

Current

```java
patientethnicityDetails.get(0);
```

Recommended

```java
patientethnicityDetails = em.createQuery(cq).getResultList();

if (patientethnicityDetails == null || patientethnicityDetails.isEmpty()) {
    return "";
}

return patientethnicityDetails.get(0);
```

Same fix required for

```
getPatientGranularRace()
```

---

# Validation

Validation performed

- ✔ quality_measures_patient_entries exists
- ✔ Provider verified
- ✔ TIN verified
- ✔ Patient insurance verified
- ✔ Not Met calculation verified
- ✔ billinglookup lookup verified
- ✔ Patient demographic values verified

---

# Risks

If only DB is corrected:

- Future invalid demographic codes may trigger the same issue.

If only code is fixed:

- Missing demographic reference data remains undetected.

Recommended:

- Fix both the data and the code.

---

# Deployment Considerations

- Validate demographic reference data (`billinglookup`) after deployment.
- Ensure patient demographic codes align with supported code values.
- Deploy defensive code changes to avoid runtime exceptions.

---

# Pending Work

- Add null/empty checks in `getPatientGranularRace()`.
- Add null/empty checks in `getPatientGranularethnicity()`.
- Log missing demographic lookup codes for easier diagnosis.
- Audit existing patient records for unsupported granular race/ethnicity codes.
- Verify that `2137-8` is the correct standard code before applying broadly.

---

# Lessons Learned

- A frontend `JSON.parse` error can originate from a backend exception that results in an empty response.
- Do not assume lookup tables always contain the requested reference values.
- Always validate `getResultList()` before accessing index `0`.
- Preserve backend exception details in logs; swallowing exceptions obscures the true cause.
- Include validation of master/reference data as part of production troubleshooting.
- Data corrections can restore functionality, but defensive coding is necessary to prevent recurrence.

---

# Conclusion

The investigation traced the issue from a browser-side JSON parsing error to the backend patient detail generation flow. Database validation confirmed that measure, provider, TIN, and patient records were correct. The key finding was a mismatch between the patient's granular ethnicity code and the available reference data in `billinglookup`, combined with helper methods that assumed lookup results were always present.

The immediate workaround was to correct the patient's granular ethnicity code:

```sql
UPDATE patient_registration
SET patient_registration_granular_ethnicity_code=',2137-8'
WHERE patient_registration_id=2739;
```

For a permanent solution, the application should defensively handle missing lookup values and log them for investigation, while ensuring that patient demographic reference data remains synchronized with the supported lookup codes.
