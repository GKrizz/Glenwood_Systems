
# README: Investigation of A1C Reporting CPT-II Code Saved with Next Day Date

---

# Overview

This document captures the investigation performed for the issue reported regarding **A1C Reporting CPT-II (M137x) codes** being generated with the **next day's Date of Service (DOS)** instead of the original encounter/lab result date.

The investigation focused on tracing the complete execution flow from the legacy Lab module (`SaveInvestigationModel`) to the Spring API (`Charges/saveCptIICodes`) responsible for creating the CPT-II reporting codes.

This document serves as:

- Technical Knowledge Base
- Root Cause Analysis (Investigation Phase)
- Developer Reference
- Future Troubleshooting Guide

---

# Background / Context

## Customer Issue

Email Subject:

> **MIM A1C reporting code**

Customer Question:

> **Why does the reporting code always populate the next day of the service?**

The billing history showed:

| Encounter Date | Reporting Code Date |
|---------------|---------------------|
| 04/14/2026 | 04/15/2026 |
| 05/19/2026 | 05/20/2026 |

Instead of attaching the reporting CPT-II code to the same Date of Service, the application generated it on the following day.

---

# Problem Statement

A1C quality reporting CPT-II codes (M1371 / M1372 / M1373) are being created with a billing/service date one day later than the original patient encounter.

Expected:

```
Encounter
05/19/2026

↓

M1372
05/19/2026
```

Actual:

```
Encounter
05/19/2026

↓

M1372
05/20/2026
```

---

# Impact

## Functional Impact

- Incorrect Date of Service for reporting CPT-II codes.
- Billing history becomes inconsistent.
- Users assume reporting logic is incorrect.

## Business Impact

Quality reporting codes should reflect the original encounter/lab result date.

Incorrect dates may affect:

- Audit history
- Quality reporting validation
- Customer confidence

---

# Environment Details

Legacy Application

```
glacelegacy_master
```

Primary file investigated:

```
/home/software/git/glacelegacy_master/WEB-INF/src/com/glenwood/hcare/actions/chart/labsandtests/SaveInvestigationModel.java
```

---

# System Components Involved

## Legacy Module

```
SaveInvestigationModel
```

Method

```
SaveParameters(...)
```

Responsible for

- Saving Lab Parameters
- Detecting HbA1c results
- Triggering CPT-II generation

---

## Spring API

```
Charges/saveCptIICodes
```

Status

> Investigation pending

---

# Investigation Timeline

## Step 1

Issue reported through email.

Observation:

A1C reporting CPT-II codes are created with the next day's date.

---

## Step 2

Identified possible API

```
Charges/saveCptIICodes
```

Hypothesis:

Date may be assigned inside this API.

---

## Step 3

Trace where API is called.

Located in

```
SaveInvestigationModel.java
```

Method

```
SaveParameters(...)
```

---

## Step 4

Verified trigger condition

Only HbA1c LOINC codes invoke the API.

```java
List<String> hemoglobinLoincCodes =
Arrays.asList(
"17855-8",
"17856-6",
"4548-4",
"4549-2",
"96595-4"
);
```

If

```java
hemoglobinLoincCodes.contains(labparamCode)
```

Then

```
Charges/saveCptIICodes
```

is invoked.

---

## Step 5

Traced date source.

Found:

```java
paramDate =
param[paramValueMap("PARAM_DATE")]
```

Meaning

The method does **NOT** generate today's date.

Instead,

```
PARAM_DATE
↓

paramDate
↓

parsedDate
↓

formattedDate

↓

encodedDate
```

---

## Step 6

Verified date conversion.

Code:

```java
parsedDate = sdf.parse(paramDate);

formattedDate =
outputFormat.format(parsedDate);

encodedDate =
URLEncoder.encode(formattedDate,"UTF-8");
```

Observation

No date manipulation exists.

No

```
Calendar.add()
```

No

```
plusDays(1)
```

No

```
CURRENT_DATE
```

No

```
new Date()
```

used during formatting.

---

## Step 7

Verified request sent to Spring API.

```java
String urlData =
"Charges/saveCptIICodes"
+ "?encounterId="+encounterId
+ "&date="+encodedDate
+ "&patientId="+tmpPatientId
+ "&measureIds=1"
+ "&chartid="+chartId
+ "&result="+paramValue
+ "&username="+username;
```

Observation

Legacy application explicitly passes

```
date=<encodedDate>
```

to Spring API.

---

# Root Cause Analysis

## Current Status

Root Cause NOT yet confirmed.

Current investigation indicates:

### Legacy Module

No evidence found that Legacy changes the encounter date.

The following operations are performed:

- Read PARAM_DATE
- Parse
- Format
- URL Encode
- Send to API

No day increment exists.

---

## Most Likely Investigation Area

```
Charges/saveCptIICodes
```

Potential possibilities

### Possibility 1

API ignores incoming date

and uses

```
new Date()

or

LocalDate.now()

or

CURRENT_DATE
```

while inserting services.

---

### Possibility 2

API receives correct date

but downstream service insertion uses

```
now()
```

instead of request parameter.

---

# Detailed Technical Findings

## Trigger Condition

Only HbA1c Lab Parameters

LOINC Codes

```
17855-8
17856-6
4548-4
4549-2
96595-4
```

trigger CPT-II generation.

---

## Date Source

```
PARAM_DATE
```

Read from incoming lab parameter.

---

## Date Conversion

Supported formats

```
MM/dd/yyyy HH:mm

MM/dd/yyyy HH:mm:ss

yyyy-MM-dd HH:mm:ss

yyyy-MM-dd HH:mm

yyyyMMddHHmmss

yyyyMMddHHmm
```

Converted into

```
yyyy-MM-dd HH:mm:ss
```

Then URL encoded.

---

## API Request

Generated URL

```
Charges/saveCptIICodes

encounterId

date

patientId

measureIds

chartid

result

username
```

---

# SQL Analysis and Scripts

## Investigation Queries

No SQL queries were executed during this investigation.

Database lookups observed in code:

Retrieve Encounter

```sql
SELECT lab_entries_encounter_id
FROM lab_entries
WHERE lab_entries_testdetail_id = ?
AND lab_entries_chartid = ?;
```

Retrieve Provider

```sql
SELECT encounter_service_doctor
FROM encounter
WHERE encounter_id = ?;
```

Retrieve Patient

```sql
SELECT chart_patientid
FROM chart
WHERE chart_id = ?;
```

Retrieve HEDIS Configuration

```sql
SELECT hedis_configuration_isactive
FROM hedis_configuration
WHERE hedis_configuration_reporting_year = ?;
```

Retrieve Provider Measure Configuration

```sql
SELECT
string_agg(
hedis_measure_provider_configuration_measure_id,
','
)
FROM
hedis_measure_provider_configuration
WHERE
hedis_measure_provider_configuration_provider_id = ?
AND
hedis_measure_provider_configuration_reporting_year = ?;
```

Purpose

Determine whether Measure 1 (HbA1c) is configured.

---

# Code Analysis

## File

```
SaveInvestigationModel.java
```

Method

```
SaveParameters(...)
```

Responsibilities

- Save Lab Parameters
- Detect HbA1c LOINC
- Convert date
- Invoke Spring API

No logic modifies the date.

---

# Debug Logging Added

Recommended temporary logs

```java
System.out.println("========================================");
System.out.println("[A1C] Original PARAM_DATE : " + paramDate);

System.out.println("[A1C] parsedDate     : " + parsedDate);
System.out.println("[A1C] formattedDate : " + formattedDate);
System.out.println("[A1C] encodedDate   : " + encodedDate);
System.out.println("[A1C] reportingYear : " + reportingYear);

System.out.println("[A1C] URL : " + urlData);
```

Purpose

Verify

- Original lab result date
- Parsed date
- Formatted date
- Encoded date
- API request

before calling

```
Charges/saveCptIICodes
```

---

# Validation Strategy

## Verify Console Logs

Expected

```
[A1C] Original PARAM_DATE :
05/19/2026 09:30

[A1C] formattedDate :
2026-05-19 09:30:00

[A1C] URL :
Charges/saveCptIICodes
...
date=2026-05-19+09%3A30%3A00
```

If logs show the correct date,

Legacy is functioning correctly.

Investigation should continue in Spring API.

---

## Debugger Validation

Breakpoint recommendation

```
if(hemoglobinLoincCodes.contains(labparamCode))
```

and

```
String urlData =
```

Inspect variables

```
paramDate

parsedDate

formattedDate

encodedDate

urlData
```

---

# Risks

Current investigation does NOT prove

- Spring API respects incoming date
- Database insert uses request date

Further investigation required.

---

# Pending Investigation

The following component still requires debugging.

```
Charges/saveCptIICodes
```

Items to verify

- Is request parameter `date` received?
- Is request parameter ignored?
- Is `CURRENT_DATE` used?
- Is `new Date()` used?
- Which database table stores the CPT-II service?
- Which column stores the service date?
- Does downstream service creation overwrite the provided date?

---

# Lessons Learned

## Debugging Approach

Instead of assuming the date was changed in Legacy,

the investigation traced the complete execution path.

This established that:

```
PARAM_DATE
↓

formattedDate

↓

encodedDate

↓

saveCptIICodes
```

is the actual flow.

No date increment exists within `SaveInvestigationModel`.

## Logging Recommendation

Temporary logging around date parsing and API invocation is valuable for confirming whether the correct encounter date leaves the legacy module before investigating downstream services.

---

# Current Investigation Status

| Component | Status |
|----------|--------|
| SaveInvestigationModel | ✅ Investigated |
| PARAM_DATE Source | ✅ Identified |
| Date Formatting | ✅ Verified |
| URL Generation | ✅ Verified |
| Legacy Date Manipulation | ❌ Not Found |
| saveCptIICodes | ⏳ Pending Investigation |
| Database Insert Logic | ⏳ Pending Investigation |

---

# Next Steps

1. Debug `Charges/saveCptIICodes`.
2. Verify whether the `date` request parameter is read and used.
3. Trace the service/CPT-II insert logic.
4. Identify the database table and date column used for persistence.
5. Confirm whether the insert uses the provided encounter date or the current system date.
