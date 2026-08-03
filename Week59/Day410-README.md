
# Day 410 - Case #248601 - VAIDYA MACRA Tab ECQM Failure Investigation

---

# Overview

This document captures the complete investigation performed for **Case #248601** related to the **VAIDYA MACRA Tab** where ECQM Measure Status failed to load for a patient.

The document serves as:

- Technical Knowledge Base
- Root Cause Analysis (RCA)
- Troubleshooting Guide
- Support Documentation
- Developer Onboarding Reference
- Future Maintenance Guide

---

# Background / Context

The MACRA module validates patient Quality Measures (eCQM) by sending patient QDM data to the centralized ECQM validation service.

When opening the MACRA tab for the patient, the frontend invokes the backend API:

```
GET
/api/desktop/user/QPPPerformance/getCQMStatusByPatient
```

The backend performs the following operations:

1. Determine Group vs Individual Reporting.
2. Load Provider Configuration.
3. Load configured Quality Measures.
4. Build QDM Request Object.
5. Call ECQM Validation Hub.
6. Receive Measure Status.
7. Save Measure Status into Patient Entries.
8. Return Measure Status to UI.

---

# Problem Statement

For

```
Patient ID : 15725
Reporting Year : 2026
Account : vaidya
```

Opening the MACRA tab failed.

Frontend displayed

```
Unable to get ECQM Measure Status
```

Backend API returned

```json
{
    "login": true,
    "success": false,
    "isAuthorizationPresent": true,
    "canUserAccess": true,
    "data": null,
    "errorMessage": "No content to map due to end-of-input"
}
```

HTTP Status

```
500 Internal Server Error
```

---

# Impact

## Functional Impact

- ECQM Measure Status could not be calculated.
- MACRA screen failed to load.
- Patient quality measure information was unavailable.

## Technical Impact

Since ECQM validation failed,

- Measure Status was never returned.
- Measure Details were never persisted.
- Downstream ECQM JSON generation failed.
- Download JSP later attempted to access a non-existent JSON file.

---

# Environment Details

| Item | Value |
|------|------|
| Account | vaidya |
| Patient | 15725 |
| Reporting Year | 2026 |
| Shared Folder | /mnt/vs23bshared/vaidya/ |
| Backend | glaceemr_backend_new |
| Spring URL | https://dev-springs.glaceemr.com/glaceemr_backend_stable_v2 |
| Gateway URL | https://hub-glacecds.glaceemr.com/glacecds/ECQMServices/validateECQM |

---

# System Components Involved

## Controller

```
QPPPerformanceController.java
```

Method

```java
@RequestMapping("/getCQMStatusByPatient")
```

---

## Services

```
MeasureCalcServiceImpl
```

Responsible for

- Group/Individual determination
- QDM Request creation
- Saving Measure Details

---

```
QPPConfServiceImpl
```

Responsible for

- Provider Configuration
- Measure Mapping

---

## Utility

```
EMeasureUtils.java
```

Responsible for

- Loading Measure Metadata
- Loading Value Sets
- Generating ECQM JSON

---

## HTTP Utility

```
HttpConnectionUtils.java
```

Responsible for

- Calling ECQM Validation Hub

---

# Investigation Timeline

## Step 1

Frontend returned

```
HTTP 500
```

with

```
No content to map due to end-of-input
```

Initial suspicion:

- Empty JSON
- Invalid Response
- Backend parsing failure

---

## Step 2

Controller reviewed.

Flow identified.

```text
getCQMStatusByPatient()

↓

checkGroupOrIndividual()

↓

getCompleteProviderInfo()

↓

getMeasureBeanDetails()

↓

Create Request

↓

POST validateECQM

↓

Parse Response

↓

Save Measures
```

---

## Step 3

Important parsing statement identified.

```java
responseFromCentralServer =
objectMapper.readValue(responseStr, Response.class);
```

This is the location producing

```
No content to map due to end-of-input
```

---

## Step 4

Reviewed HTTP Utility.

```
HttpConnectionUtils.postData()
```

Current implementation

```java
reader =
new BufferedReader(
new InputStreamReader(conn.getInputStream()));
```

Observation

The implementation only reads

```
getInputStream()
```

It never reads

```
getErrorStream()
```

If validation server returns

```
HTTP 400

or

HTTP 500
```

actual response body is discarded.

This makes debugging difficult.

---

## Step 5

Gateway URLs verified.

MACRA Validation

```sql
SELECT *
FROM gateway_controller
WHERE gateway_controller_module_name =
'MACRA measure validation';
```

Result

```
https://hub-glacecds.glaceemr.com/glacecds/ECQMServices/validateECQM
```

---

ICD10 Gateway

```sql
SELECT *
FROM gateway_controller
WHERE gateway_controller_module_name =
'icd10 for eCQM Services';
```

Returned correctly.

---

## Step 6

Session verified.

Important session variables

```
sharedFolderPath

=

/mnt/vs23bshared/vaidya/
```

No session issue observed.

---

## Step 7

Secondary exception discovered.

```
downloadAndDeleteTemp.jsp
```

Stacktrace

```
FileNotFoundException

/mnt/vs23bshared/vaidya/
MU/
ECQMPatientObjects/
.json
```

Observation

Filename itself was empty.

Instead of

```
ECQM_xxx.json
```

application attempted to open

```
.json
```

This indicates ECQM JSON was never generated.

---

# Root Cause Analysis

## Confirmed Root Cause

The immediate backend exception occurs during JSON deserialization.

```java
objectMapper.readValue(responseStr, Response.class);
```

Jackson throws

```
No content to map due to end-of-input
```

only when

```
responseStr

is

NULL

or

EMPTY
```

Therefore,

the ECQM Validation Hub did not provide a valid JSON response.

---

## Likely Upstream Cause

The exact upstream reason was **not confirmed during this investigation**.

Possible causes include:

- Empty response from ECQM validation service.
- HTTP error response from validation service that is hidden because `HttpConnectionUtils` only reads `getInputStream()`.
- Invalid or incomplete request payload (e.g., missing measures or malformed request).
- Failure in downstream validation service (`validateECQM`).

Further instrumentation is required to identify the precise upstream failure.

---

## Secondary Failure

Because no valid ECQM response was received:

- ECQM JSON file was not generated.
- Download JSP attempted to download a file with an empty filename.
- Resulted in:

```
FileNotFoundException

/mnt/vs23bshared/vaidya/MU/ECQMPatientObjects/.json
```

This is a downstream symptom, **not the primary cause**.

---

# Detailed Technical Findings

## API

```
GET

/api/desktop/user/QPPPerformance/getCQMStatusByPatient
```

---

## External Validation Service

```
https://hub-glacecds.glaceemr.com/glacecds/ECQMServices/validateECQM
```

---

## Supporting Gateway

```
https://datagateway.glaceemr.com/DataGatewayMediSpan/eCQMServices
```

---

## Database Tables

### gateway_controller

Stores external gateway URLs.

---

### macra_configuration

Stores

- Reporting Year
- Group/Individual configuration

---

### macra_provider_configuration

Stores

- Reporting Start
- Reporting End
- Provider Configuration

---

### quality_measures_provider_mapping

Stores provider-measure mapping.

---

### quality_measures_patient_entries

Stores patient ECQM calculation results.

---

### initial_settings

Used for

```
MIPS Report By Billing Doctor
```

---

# SQL Analysis and Scripts

## Investigation Queries

### Gateway Validation

```sql
SELECT *
FROM gateway_controller
WHERE gateway_controller_module_name =
'MACRA measure validation';
```

Purpose

- Validate ECQM Hub URL.

---

### ICD10 Gateway

```sql
SELECT *
FROM gateway_controller
WHERE gateway_controller_module_name =
'icd10 for eCQM Services';
```

Purpose

- Validate ICD10 lookup service.

---

### Initial Setting Validation

```sql
SELECT
initial_settings_option_name,
initial_settings_option_value
FROM initial_settings
WHERE initial_settings_option_name =
'MIPS Report By Billing Doctor';
```

Purpose

- Verify reporting configuration.

---

### Provider Configuration Validation

Recommended query

```sql
SELECT *
FROM macra_provider_configuration
WHERE
macra_provider_configuration_provider_id = 1
AND
macra_provider_configuration_reporting_year = 2026;
```

Purpose

- Validate provider reporting configuration.

---

### Provider Measure Mapping

Recommended query

```sql
SELECT *
FROM quality_measures_provider_mapping
WHERE
quality_measures_provider_mapping_provider_id = 1
AND
quality_measures_provider_mapping_reporting_year = 2026;
```

Purpose

- Validate configured ECQM measures.

---

# Code Analysis

## Controller

```
QPPPerformanceController
```

Critical statement

```java
String responseStr =
HttpConnectionUtils.postData(...);

responseFromCentralServer =
objectMapper.readValue(responseStr,
Response.class);
```

Risk

Fails if

```
responseStr == ""
```

---

## HttpConnectionUtils

Current implementation

```java
reader =
new BufferedReader(
new InputStreamReader(conn.getInputStream()));
```

Issue

Never reads

```
conn.getErrorStream()
```

Consequences

- Actual server-side error body is lost.
- Difficult debugging.
- Generic Jackson exception instead of meaningful validation error.

---

## EMeasureUtils

Responsible for

```
getMeasureBeanDetails()

getJSONFile()
```

Generates ECQM JSON under

```
/MU/ECQMPatientObjects/
```

Failure in upstream validation prevents JSON generation.

---

# Recommended Code Improvements

## 1. Log Request

```java
LOGGER.info("Request={}", requestString);
```

---

## 2. Log Response

```java
LOGGER.info("Response={}", responseStr);
```

---

## 3. Validate Empty Response

```java
if(responseStr == null ||
responseStr.trim().isEmpty()){

    throw new RuntimeException(
        "Empty response received from ECQM Validation Hub");

}
```

---

## 4. Read Error Stream

Instead of

```java
conn.getInputStream();
```

Use

```java
int status = conn.getResponseCode();

InputStream stream =
status >= 400 ?
conn.getErrorStream()
:
conn.getInputStream();
```

Benefit

Actual validation service errors become visible.

---

# Validation and Testing

## UI Validation

Open

```
MACRA Tab
```

Expected

Measures should load.

---

## API Validation

Verify

```
HTTP 200
```

Expected JSON

```json
{
    "measureStatus": ...
}
```

Not

```
Empty
```

---

## Backend Logs

Verify

```
Request JSON

Response JSON

HTTP Status
```

---

## SQL Validation

Verify

- Provider configuration exists.
- Measure mapping exists.
- Initial setting exists.
- Gateway URLs are configured correctly.

---

# Risks and Side Effects

## Risks

- Empty response causes complete ECQM failure.
- Missing logging delays troubleshooting.
- Hidden HTTP error responses obscure root cause.
- Downstream file generation silently fails.

---

## Data Integrity

No evidence of data corruption.

Failure occurs before patient measure persistence.

---

## Backward Compatibility

Adding

- request logging
- response logging
- error stream handling
- null validation

is backward compatible.

---

# Pending Work

The following items remain unresolved:

1. Confirm actual HTTP status returned by `validateECQM`.
2. Capture raw response body from ECQM Hub.
3. Verify generated `requestString` contents.
4. Verify provider measure list is populated.
5. Verify `MIPS Report By Billing Doctor` setting value.
6. Determine why ECQM JSON file generation was skipped.

---

# Lessons Learned

## Technical Insights

- `No content to map due to end-of-input` almost always indicates an empty or null payload passed to Jackson's `ObjectMapper.readValue()`.
- Downstream errors (such as missing files) can be symptoms rather than the root cause.
- HTTP clients should inspect both `getInputStream()` and `getErrorStream()` to preserve server-side diagnostics.
- Logging the outbound request, inbound response, and HTTP status significantly reduces debugging time.

## Prevention Recommendations

- Add defensive checks before JSON deserialization.
- Enhance `HttpConnectionUtils` to log HTTP status codes and error bodies.
- Include correlation IDs or request identifiers when invoking external ECQM services.
- Emit explicit logs when ECQM JSON generation is skipped due to upstream failures.
- Add health checks or monitoring for the external `validateECQM` service to detect outages or empty responses proactively.
