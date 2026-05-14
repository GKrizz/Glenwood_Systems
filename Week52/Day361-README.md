# Case# 245797 - NJAVVAJI --- MIPS Dashboard JSON Parsing Failure – Root Cause Analysis and Technical Resolution

## Overview

This document captures the complete investigation, debugging process, root cause analysis, code review, database validation, workaround implementation, and permanent fix for a production issue where the MIPS Dashboard failed to load for the `njavvaji` account.

The issue initially appeared as a frontend JSON parsing error but was ultimately traced to a missing database configuration entry in the `gateway_controller` table.

This README serves as:

* Technical RCA document
* Debugging reference
* Production support guide
* Developer onboarding reference
* Long-term maintenance documentation
* Knowledge base article

---

# Background / Context

The MIPS Dashboard feature retrieves MIPS/eCQM performance data through backend Spring APIs and renders it in the frontend UI.

The issue was reported specifically for:

* Account: `njavvaji`
* Module: MIPS Dashboard
* Reporting Year: `2026`

Other accounts such as:

* `dbc`
* `calvary`
* `mim`

were functioning correctly.

---

# Problem Statement

The MIPS Dashboard failed to load and displayed a frontend JSON parsing error.

Frontend error:

```javascript
SyntaxError: JSON.parse: unexpected end of data at line 2 column 1 of the JSON data
```

Backend logs showed:

```text
org.json.JSONException: A JSONObject text must begin with '{' at character 0
```

and later:

```text
java.lang.ArrayIndexOutOfBoundsException:
Index 1 out of bounds for length 1
```

The API response became invalid because backend processing failed before generating a proper JSON response.

---

# Impact

## Business Impact

* Providers could not view MIPS Dashboard reports.
* Dashboard UI failed completely.
* Performance data visibility was blocked.
* Production support escalation was required.

## Technical Impact

* Backend transaction rollback occurred.
* Incomplete HTTP response returned.
* Frontend received malformed/empty JSON.
* JSON parsing failed in browser.

---

# Environment Details

| Component        | Value                              |
| ---------------- | ---------------------------------- |
| Module           | MIPS Dashboard                     |
| API              | QPPPerformance/getDashBoardDetails |
| Reporting Year   | 2026                               |
| Environment      | Production-like                    |
| Affected Account | njavvaji                           |
| Working Accounts | dbc, calvary, mim                  |
| Framework        | Spring Boot                        |
| Database         | PostgreSQL                         |
| UI Error         | JSON.parse unexpected end of data  |

---

# System Components Involved

## APIs

### Dashboard API

```text
QPPPerformance/getDashBoardDetails
```

Example URL:

```text
http://172.18.24.92/glaceemr_backend_stable_v2/njavvaji/api/desktop/user/QPPPerformance/getDashBoardDetails
```

---

## Java Classes

| Class                      | Responsibility              |
| -------------------------- | --------------------------- |
| `QPPPerformanceController` | Dashboard API controller    |
| `MeasureCalcServiceImpl`   | Core MIPS calculation logic |
| `CustomURLConnection`      | Backend HTTP connector      |
| `MIPSPerformanceBean`      | Dashboard result object     |
| `EMRResponseBean`          | API response wrapper        |

---

## Important Methods

| Method                        | Purpose                             |
| ----------------------------- | ----------------------------------- |
| `getDashBoardDetails()`       | Dashboard data generation           |
| `getMeasureRateReportByNPI()` | Builds performance report           |
| `getCMSIdAndTitle()`          | Retrieves CMS measure title/details |
| `getHubServerUrl()`           | Reads gateway configuration         |

---

## Database Tables

| Table                               | Purpose                        |
| ----------------------------------- | ------------------------------ |
| `gateway_controller`                | External service configuration |
| `macra_measures_rate`               | Measure performance data       |
| `quality_measures_provider_mapping` | Measure mapping configuration  |
| `quality_measures_patient_entries`  | Patient-level measure entries  |

---

# Investigation Timeline

## Phase 1 – Initial UI Error

Frontend displayed:

```javascript
SyntaxError: JSON.parse: unexpected end of data
```

Initial assumption:

* Invalid JSON returned from backend.

---

## Phase 2 – Backend Response Inspection

Debug logs added to:

```java
CustomURLConnection.perform()
```

Observed:

```text
RAW=[]
TRIM=[]
LENGTH=[0]
```

Meaning:

* Backend returned empty response body.

---

## Phase 3 – Authentication Investigation

Observed:

```text
ResponseCode=401
```

Authentication/token logic reviewed.

`access_token` handling and authorization headers were validated.

This was NOT the final root cause.

---

## Phase 4 – Exception Stack Trace Analysis

Critical exception discovered:

```text
java.lang.ArrayIndexOutOfBoundsException:
Index 1 out of bounds for length 1
```

Location:

```java
MeasureCalcServiceImpl.getMeasureRateReportByNPI()
```

Failing line:

```java
resultObject.setTitle(cmsIdNTitle.split("&&&")[1]);
```

---

## Phase 5 – Debug Logging Added

Detailed sysouts were added:

```java
System.out.println("cmsIdNTitle = [" + cmsIdNTitle + "]");
```

Observed:

```text
cmsIdNTitle = []
cmsSplit.length = 1
```

Meaning:

* Expected format:

```text
CMS349v8&&&Title
```

* Actual format:

```text
""
```

---

## Phase 6 – getCMSIdAndTitle() Investigation

Method analyzed:

```java
private String getCMSIdAndTitle(String measureId, String accountId,Integer year)
```

Method attempted:

1. Read local JSON file
2. Else call remote API
3. Build:

```text
cmsId&&&title
```

Observed:

```text
FINAL CMS RESULT = []
```

Thus:

* CMS metadata retrieval completely failed.

---

## Phase 7 – Gateway Configuration Investigation

Investigated:

```java
getHubServerUrl()
```

Query:

```sql
SELECT *
FROM gateway_controller
WHERE gateway_controller_module_name =
'icd10 for eCQM Services';
```

Result in `njavvaji`:

```text
(0 rows)
```

Result in working accounts:

```text
gateway_controller_module_name = 'icd10 for eCQM Services'
```

This identified the actual root cause.

---

# Root Cause Analysis

## Actual Root Cause

The `gateway_controller` configuration entry:

```text
icd10 for eCQM Services
```

was missing in the `njavvaji` database.

Because of this:

1. `getHubServerUrl()` returned empty results
2. CMS API URL generation failed
3. `getCMSIdAndTitle()` returned empty string
4. `cmsIdNTitle.split("&&&")[1]` crashed
5. Transaction rolled back
6. API returned incomplete JSON
7. Frontend JSON parsing failed

---

## Technical Failure Chain

```text
Missing gateway_controller entry
        ↓
getHubServerUrl() failed
        ↓
CMS API URL unavailable
        ↓
getCMSIdAndTitle() returned ""
        ↓
split("&&&")[1] failure
        ↓
ArrayIndexOutOfBoundsException
        ↓
Transaction rollback
        ↓
Invalid JSON response
        ↓
Frontend JSON.parse error
```

---

# Detailed Technical Findings

## Failure Point

### Unsafe Code

```java
resultObject.setCmsId(cmsIdNTitle.split("&&&")[0]);
resultObject.setTitle(cmsIdNTitle.split("&&&")[1]);
```

Problem:

* Assumed split always returns 2 elements.

---

## Defensive Fix Added

```java
String[] cmsSplit = cmsIdNTitle.split("&&&");

if(cmsSplit.length > 1){
    resultObject.setCmsId(cmsSplit[0]);
    resultObject.setTitle(cmsSplit[1]);
}else{
    resultObject.setCmsId("");
    resultObject.setTitle("");
}
```

---

## Important Clarification

This code change:

* DOES NOT update database
* DOES NOT modify persisted data
* ONLY changes runtime response object
* ONLY affects API response/UI rendering

Safe for production deployment.

---

# SQL Analysis and Scripts

# Investigation Queries

## Check Missing Gateway Configuration

```sql
SELECT *
FROM gateway_controller
WHERE gateway_controller_module_name =
'icd10 for eCQM Services';
```

### Purpose

Validate whether required gateway configuration exists.

### Type

Investigation Query

---

## Compare with Working Accounts

Executed in:

* dbc
* calvary
* mim

Observed valid rows.

---

## Validate Measure Mapping

```sql
SELECT *
FROM quality_measures_provider_mapping
WHERE quality_measures_provider_mapping_measure_id IN
('155','475','488');
```

### Purpose

Verify provider measure mapping configuration.

### Type

Investigation Query

---

## Validate MIPS Measure Data

```sql
SELECT *
FROM macra_measures_rate
WHERE macra_measures_rate_measure_id IN
('155','475','488')
AND macra_measures_rate_reporting_year = 2026;
```

### Purpose

Validate generated numerator/denominator data.

### Findings

* Data existed
* Some measures had 0 denominator/numerator
* Not root cause

---

# Fix Script

## Permanent Database Fix

```sql
INSERT INTO gateway_controller
(
    gateway_controller_id,
    gateway_controller_module_name,
    gateway_controller_url
)
VALUES
(
    12,
    'icd10 for eCQM Services',
    'https://datagateway.glaceemr.com/DataGatewayMediSpan/eCQMServices'
);
```

### Type

Configuration Fix

### Impact

Restored CMS metadata retrieval.

### Risk

Low.

Validated against working environments.

---

# Rollback Script

```sql
DELETE
FROM gateway_controller
WHERE gateway_controller_id = 12;
```

---

# Code Changes

## File

```text
MeasureCalcServiceImpl.java
```

---

## Changes Added

### Defensive Split Validation

Before:

```java
resultObject.setCmsId(cmsIdNTitle.split("&&&")[0]);
resultObject.setTitle(cmsIdNTitle.split("&&&")[1]);
```

After:

```java
String[] cmsSplit = cmsIdNTitle.split("&&&");

if(cmsSplit.length > 1){
    resultObject.setCmsId(cmsSplit[0]);
    resultObject.setTitle(cmsSplit[1]);
}else{
    resultObject.setCmsId("");
    resultObject.setTitle("");
}
```

---

## Additional Debug Logging

Added:

```java
System.out.println("cmsIdNTitle = [" + cmsIdNTitle + "]");
System.out.println("cmsSplit.length = " + cmsSplit.length);
```

---

# Fixes and Workarounds

# Temporary Workaround

Defensive null/split handling prevented backend crash.

---

# Permanent Fix

Insert missing configuration row:

```text
gateway_controller_module_name =
'icd10 for eCQM Services'
```

---

# Validation and Testing

## Validation Steps

### 1. Verify Configuration Exists

```sql
SELECT *
FROM gateway_controller
WHERE gateway_controller_module_name =
'icd10 for eCQM Services';
```

---

### 2. Re-run Dashboard API

Confirmed:

* API returned valid JSON
* No transaction rollback

---

### 3. Validate UI

Confirmed:

* MIPS Dashboard loads successfully
* No JSON.parse error

---

### 4. Log Validation

Confirmed removal of:

```text
ArrayIndexOutOfBoundsException
```

and:

```text
JSON.parse unexpected end of data
```

---

# Risks and Side Effects

## Production Risks

### Minimal

Configuration row already existed in other environments.

---

## Code Risks

Defensive split handling may mask future configuration issues if logs are ignored.

Recommendation:

* Keep warning logs enabled.

---

# Pending Work

## Recommended Improvements

### 1. Add Proper Null Validation

`getCMSIdAndTitle()` should never return empty silently.

---

### 2. Improve Exception Handling

Instead of:

```java
catch(Exception e){}
```

Use structured logging.

---

### 3. Add Monitoring

Alert when:

```text
gateway_controller lookup returns 0 rows
```

---

### 4. Improve API Error Response

Backend should return structured JSON error instead of partial response.

---

# Lessons Learned

## Debugging Insights

### Frontend JSON errors may originate from backend transaction failures.

---

## Configuration Dependencies Matter

Missing configuration tables can indirectly break unrelated modules.

---

## Avoid Unsafe Array Splits

Never assume:

```java
split()[1]
```

exists.

Always validate length.

---

## Add Defensive Programming

Production APIs should not crash because of missing metadata.

---

## Improve Logging

The following debug logs were critical:

```java
System.out.println("cmsIdNTitle = " + cmsIdNTitle);
```

Without these logs, identifying the empty response source would have taken much longer.

---

# Final Resolution Summary

## Root Cause

Missing configuration row in:

```text
gateway_controller
```

for:

```text
icd10 for eCQM Services
```

---

## Permanent Resolution

Inserted missing configuration:

```sql
INSERT INTO gateway_controller
(
    gateway_controller_id,
    gateway_controller_module_name,
    gateway_controller_url
)
VALUES
(
    12,
    'icd10 for eCQM Services',
    'https://datagateway.glaceemr.com/DataGatewayMediSpan/eCQMServices'
);
```
