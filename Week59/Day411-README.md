
# QRDA-I Export Investigation – Patient Principal Doctor Resolution & QRDA Generation Debugging (Measure 134)

---

# Overview

This document captures the complete technical investigation performed while debugging QRDA-I export generation for **CMS Measure 134**.

The investigation primarily focused on:

- QRDA-I export flow
- Patient filtering
- Principal Doctor resolution
- Patient metadata retrieval
- Hibernate queries
- PostgreSQL validation
- Entity mapping investigation
- QRDA XML generation workflow

This README is intended to serve as:

- Technical knowledge base
- Root Cause Analysis (RCA)
- Troubleshooting guide
- Future maintenance reference
- Developer onboarding documentation

---

# Background / Context

The QRDA-I export API was invoked to generate QRDA XMLs for Measure **134**.

API:

```
POST
/pcawh/api/emr/user/QRDAController/exportQRDA
```

Request:

```json
{
  "measureId":134,
  "reportingYear":2026,
  "providerId":-1,
  "groupOrIndividual":"1",
  "category":1,
  "insideSpecificMeasure":true,
  "ssn":"204503813"
}
```

The export process:

1. Finds eligible patients
2. Applies filtering
3. Creates QDM Request
4. Generates QRDA XML
5. Creates ZIP

During debugging, investigation expanded into verifying whether patient/provider mappings or Hibernate entity mappings were causing failures.

---

# Problem Statement

The QRDA export process required verification that:

- Eligible patients were correctly selected.
- Principal doctor values were valid.
- Provider lookup succeeded.
- QRDA generation completed without failures.
- Entity mappings were compatible with PostgreSQL BIGINT columns.

Another investigation point was a previously observed Hibernate error related to retrieving the patient's principal doctor.

---

# Impact

Potential impacts included:

- QRDA generation failure.
- Missing QRDA XMLs.
- Incorrect provider mapping.
- Failure during QDM Request generation.
- Runtime Hibernate type mismatch.

---

# Environment Details

| Item | Value |
|-------|---------|
| Database | PostgreSQL |
| Account | pcawh |
| Measure | 134 |
| Reporting Year | 2026 |
| API | QRDAController/exportQRDA |
| Export Type | QRDA-I |
| Category | 1 |
| Provider Mode | All Providers |
| SSN/TIN | 204503813 |

---

# System Components Involved

## Controller

```
QRDAController
```

Method

```
exportQRDA()
```

---

## Service

```
ExportQRDAServiceImpl
```

Method

```
exportQRDAI()
```

---

## Supporting Methods

```
getUniquePatientList()

getFilteredDetails()

getProviderList()

getPatientNames()

getQDMRequestObject()

generateCDA()

countXmlFiles()
```

---

## Database Tables

### quality_measures_patient_entries

Used to determine:

- Eligible patients
- Measure status
- Reporting year
- TIN
- IPP

---

### patient_registration

Used for

- Patient demographics
- Principal doctor
- Patient name

---

### emp_profile

Used for

- Provider lookup

---

### staff_pin_number_details

Used for

- SSN/TIN → Employee mapping

---

### initial_settings

Used for

System configuration lookup.

---

# Investigation Timeline

---

## Step 1

QRDA export API executed.

Returned successfully:

```
QRDA_All_134_xxxxxxxxx
```

indicating export started.

---

## Step 2

Controller logging added.

Logged:

- request bean
- provider
- ssn
- measure
- unique patient count

---

## Step 3

Verified unique patient list.

Hibernate generated:

```sql
SELECT DISTINCT
quality_measures_patient_entries_patient_id
FROM quality_measures_patient_entries
WHERE reporting_year=2026
AND ipp>0
AND measure_id=134
AND tin='204503813'
```

Returned:

```
3259 patients
```

---

## Step 4

Filtered patient list generated.

Only:

```
26 patients
```

were selected after applying additional filters.

---

## Step 5

Verified patient IDs.

Patient list included:

```
41929
149519
149815
...
376590
...
```

---

## Step 6

Patient names loaded.

Hibernate executed:

```sql
SELECT
patient_registration_id,
patient_registration_first_name,
patient_registration_last_name
FROM patient_registration
WHERE patient_registration_id IN (...)
```

Returned successfully.

---

## Step 7

QRDA generation started.

Log:

```
Processing Patient 1/26
```

---

## Step 8

Investigation shifted toward principal doctor retrieval.

Method:

```
getPrincipleDoctor()
```

---

## Step 9

Investigated patient:

```
376590
```

Database values:

```
Principal Doctor = NULL
```

Initially suspected as root cause.

---

## Step 10

Validated PostgreSQL schema.

```
patient_registration_principal_doctor
```

type:

```
BIGINT
```

---

## Step 11

Verified account data.

Unlike previous account,

PCAWH contained valid principal doctor values.

Example:

| Patient | Principal Doctor |
|----------|------------------|
|124|803|
|126|15|
|177|15|
|190|41|

No NULL values found.

---

## Step 12

Verified employee existence.

Query:

```sql
LEFT JOIN emp_profile
```

No missing employee records.

Result:

```
0 rows
```

Meaning:

Every principal doctor existed.

---

## Step 13

Verified patients without principal doctor.

Returned:

```
0 rows
```

Thus missing provider data was ruled out.

---

## Root Cause Analysis

## Confirmed Findings

### Patient Filtering

Working correctly.

Unique patient selection worked.

---

### Provider Mapping

Working correctly.

Every patient had a valid principal doctor.

---

### Employee Lookup

Working correctly.

Every doctor exists in emp_profile.

---

### Missing Principal Doctor

NOT the cause.

---

### QRDA Export Flow

Successfully reached

```
Processing Patient 1/26
```

Therefore:

- Patient retrieval completed
- Patient name retrieval completed
- Provider lookup completed

---

## Potential Root Cause (Still Under Investigation)

A suspected Hibernate type mismatch exists.

Observed:

Database

```
patient_registration_id BIGINT

patient_registration_principal_doctor BIGINT
```

Code

```java
CriteriaQuery<Integer>
```

Possible mismatch:

```
BIGINT

↓

Long

↓

Integer expected
```

Potential issue if entity mapping does not match database type.

This investigation remained open because entity class was not yet inspected.

---

# Detailed Technical Findings

## QRDA Export Flow

```
API

↓

Controller

↓

Unique Patient List

↓

Filtered Patient List

↓

Patient Names

↓

Measure Loading

↓

QDM Request

↓

generateCDA()

↓

XML

↓

ZIP
```

---

## Logging Added

Controller

```
provider

ssn

measure

unique patient count

filtered patients
```

Service

```
Patient count

Patient list

Processing patient

Measure

CMS ID

Request creation

Intervention count

XML success

Summary
```

---

## QRDA Output Summary

Generated:

```
Total Patients

Generated XMLs

Failed XMLs

Output Folder

XML Count
```

---

# SQL Analysis and Scripts

## Investigation Query

Retrieve eligible patients.

```sql
SELECT DISTINCT
    quality_measures_patient_entries_patient_id
FROM quality_measures_patient_entries
WHERE quality_measures_patient_entries_reporting_year = 2026
  AND quality_measures_patient_entries_ipp > 0
  AND quality_measures_patient_entries_measure_id = '134'
  AND quality_measures_patient_entries_tin = '204503813';
```

Purpose

Identify all eligible patients.

---

## Validation Query

Count eligible patients.

```sql
SELECT COUNT(DISTINCT quality_measures_patient_entries_patient_id)
FROM quality_measures_patient_entries
WHERE quality_measures_patient_entries_reporting_year = 2026
  AND quality_measures_patient_entries_ipp > 0
  AND quality_measures_patient_entries_measure_id = '134'
  AND quality_measures_patient_entries_tin = '204503813';
```

Result

```
3259
```

---

## Investigation Query

Retrieve patient names.

```sql
SELECT
patient_registration_id,
patient_registration_first_name,
patient_registration_last_name
FROM patient_registration
WHERE patient_registration_id IN (...)
```

Purpose

Populate QRDA filename.

---

## Investigation Query

Verify principal doctor.

```sql
SELECT
patient_registration_id,
patient_registration_principal_doctor
FROM patient_registration
WHERE patient_registration_id=376590;
```

Purpose

Check provider assignment.

---

## Validation Query

Verify provider exists.

```sql
SELECT
ep.emp_profile_empid,
ep.emp_profile_fullname
FROM emp_profile ep
WHERE ep.emp_profile_empid=
(
SELECT patient_registration_principal_doctor
FROM patient_registration
WHERE patient_registration_id=376590
);
```

Purpose

Validate employee mapping.

---

## Investigation Query

Provider mapping verification.

```sql
SELECT
pr.patient_registration_id,
pr.patient_registration_principal_doctor
FROM patient_registration pr
LEFT JOIN emp_profile ep
ON ep.emp_profile_empid=
pr.patient_registration_principal_doctor
WHERE ...
```

Purpose

Detect orphan providers.

Result

```
0 rows
```

---

## Investigation Query

Patients without principal doctor.

```sql
SELECT
patient_registration_id
FROM patient_registration
WHERE patient_registration_principal_doctor IS NULL;
```

Purpose

Identify missing assignments.

Result

```
0 rows
```

---

# Code Analysis

## Controller

Added logging:

```java
provider

ssn

measure

filtered patients
```

---

## ExportQRDAServiceImpl

Added logging for:

```
Patient Count

Patient IDs

Processing Patient

Request Object

Intervention Count

XML Path

Summary
```

---

## getPatientName()

Purpose

Retrieve patient names.

Implementation:

```java
CriteriaQuery<Object[]>

patientRegistrationId

patientRegistrationFirstName

patientRegistrationLastName
```

Returns:

```
Map<Integer,String>
```

Format

```
FirstName&&&LastName
```

Used for:

```
QRDA_filename.xml
```

---

## Suspected Mapping Issue

Current code uses

```java
Integer
```

Database stores

```
BIGINT
```

Requires entity verification.

Potential safer mapping:

```java
Long
```

---

# Validation Performed

Completed:

- API invocation
- Hibernate SQL verification
- Patient selection
- Filter validation
- Principal doctor validation
- Employee lookup validation
- NULL provider validation
- XML generation flow verification
- Patient name retrieval validation

---

# Risks

## Entity Mapping

If entity uses Integer while DB uses BIGINT:

- Hibernate InstantiationException
- ClassCastException
- Type conversion failures

---

## Logging

Without additional logs,

failure location is difficult to identify.

---

## Data Integrity

Provider IDs must exist in

```
emp_profile
```

Otherwise QRDA generation could fail.

---

# Workarounds

Temporary

Extensive logging added.

Validated:

- patient list
- provider list
- request object
- intervention list

---

# Pending Work

## Still Pending

Inspect entity:

```
PatientRegistration.java
```

Verify

```
patientRegistrationId

patientRegistrationPrincipalDoctor
```

types.

---

Inspect

```
PatientRegistration_.java
```

Verify generated metamodel types.

---

Confirm whether

```
Integer

vs

Long
```

mapping mismatch exists.

---

# Lessons Learned

1. Always verify PostgreSQL column types before assuming Hibernate mappings.

2. A successful patient retrieval does **not** guarantee entity mappings are correct; verify Java entity types against the database schema.

3. Distinguish between data issues (e.g., NULL principal doctor, missing employee) and application issues (e.g., Hibernate type mismatches). In this investigation, the PCAWH account had valid provider data, ruling out data quality as the primary cause.

4. Add structured logging at key stages of the QRDA export pipeline:
   - Request parameters
   - Unique patient selection
   - Filtered patient list
   - Provider resolution
   - QDM request creation
   - XML generation
   - Final export summary

5. Validate database assumptions with SQL before modifying application code. Queries confirmed:
   - 3,259 eligible patients for Measure 134.
   - 26 patients after filter application.
   - No orphan `patient_registration_principal_doctor` values.
   - No patients with `NULL` principal doctors in the investigated dataset.

6. For Hibernate Criteria API, ensure the query result type (`CriteriaQuery<T>`) matches both the entity field type and the underlying database type to avoid runtime instantiation or conversion errors.

7. When debugging QRDA generation, isolate the pipeline stage first. Since execution reached `Processing Patient 1/26`, earlier stages (patient retrieval, filtering, patient name lookup) were functioning correctly, narrowing the investigation to later stages such as QDM request creation or CDA generation.
