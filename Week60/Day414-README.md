# Case Investigation README

## PQRS Tab – "Measures not configured for this Reporting Year" for Robert Filoramo DPM (Provider ID: 8)

---

# Overview

This document captures the complete investigation performed for a customer-reported issue where the **PQRS tab** displayed the message:

> **"Measures not configured for this Reporting Year"**

even though PQRS measures had already been configured for the provider.

The investigation traced the issue from the UI through the legacy JSP, Action layer, database configuration, and finally to the backend REST API responsible for supplying PQRS measure definitions.

---

# Background / Context

The customer reported that they were unable to document PQRS measures for a patient.

Customer message:

> **"Good morning - having trouble putting in our PQRS'... can you help us out?"**

The issue occurred while opening the PQRS tab from the SOAP template.

Patient Details

| Item          | Value                   |
| ------------- | ----------------------- |
| Patient ID    | **39815**               |
| Encounter ID  | **68961**               |
| Chart ID      | **39733**               |
| Provider      | **Robert Filoramo DPM** |
| Provider ID   | **8**                   |
| Encounter DOS | **07/06/2026**          |

---

# Problem Statement

While opening the PQRS tab, the application displayed:

```
Measures not configured for this Reporting Year
```

instead of displaying the configured PQRS measures.

Initially, the assumption was that no provider mapping existed for the reporting year.

However, database investigation proved that the provider **was already configured** for 2026.

---

# Impact

* Users cannot document PQRS measures.
* PQRS workflow becomes unusable.
* Claims/Registry reporting is interrupted.
* Customer assumes configuration is missing although database configuration exists.

---

# Environment Details

Legacy Application

```
glacelegacy_master
```

Primary JSP

```
jsp/chart/leafs/soap/Measurecode1.jsp
```

Action Mapping

```
WEB-INF/classes/ActionMappings.properties
```

Action Class

```
com.glenwood.hcare.actions.chart.leafs.MeasureUrl
```

Backend API

```
GlaceRulesController/getPQRSMeasure
```

---

# System Components Involved

## UI

```
Measurecode1.jsp
```

Responsible for

* Loading configured measures
* Rendering PQRS measure list
* Showing error message

---

## Action Layer

```
MeasureUrl.java
```

Responsible for

* Loading previous documented PQRS entries
* Calling backend API
* Returning JSON

---

## Backend

```
GlaceRulesController/getPQRSMeasure
```

Responsible for

* Returning configured PQRS measures

---

## Database Tables

### Configuration

```
quality_measures_provider_mapping
```

Stores configured PQRS measures.

---

### Documentation

```
pqrs_patient_entries
```

Stores documented PQRS CPT entries.

---

### Provider Configuration

```
macra_provider_configuration
```

Stores

* Reporting Method
* Reporting Year

---

### HEDIS

```
hedis_configuration
```

Determines whether HEDIS mode is enabled.

---

### Provider

```
emp_profile
```

---

### Encounter

```
encounter
```

---

# Investigation Timeline

---

## Step 1

Initial assumption

No PQRS measures configured.

Checked

```sql
SELECT *
FROM quality_measures_provider_mapping
WHERE quality_measures_provider_mapping_provider_id=8
AND quality_measures_provider_mapping_reporting_year=2026;
```

Result

Five configured measures existed.

| Measure |
| ------- |
| 113     |
| 1       |
| 134     |
| 236     |
| 112     |

Conclusion

❌ Not a configuration issue.

---

## Step 2

Checked all providers configured for 2026.

```sql
SELECT DISTINCT
quality_measures_provider_mapping_provider_id,
quality_measures_provider_mapping_reporting_year
FROM quality_measures_provider_mapping
WHERE quality_measures_provider_mapping_reporting_year=2026;
```

Result

11 providers configured.

---

## Step 3

Validated overall configuration.

```sql
SELECT COUNT(*)
FROM quality_measures_provider_mapping
WHERE quality_measures_provider_mapping_reporting_year=2026;
```

Result

```
85
```

Configuration exists.

---

## Step 4

Compared 2025 and 2026 mappings.

Provider 8 contained

2025

```
1
47
126
127
130
155
226
236
```

2026

```
1
112
113
134
236
```

Confirmed yearly mappings exist.

---

## Step 5

Verified encounter provider.

```sql
SELECT
encounter_id,
encounter_chartid,
encounter_service_doctor,
encounter_date
FROM encounter
WHERE encounter_id=68961;
```

Result

```
Service Doctor = 8
```

Confirmed UI is using correct provider.

---

## Step 6

Located UI message.

File

```
Measurecode1.jsp
```

Found

```html
Measures not configured for this Reporting Year
```

Observation

The message is static HTML.

It is **not** the decision point.

---

## Step 7

Studied JavaScript flow.

Found

```javascript
AjaxConnect.sendAsynchRequest(
    'POST',
    'MeasureURL.Action',
    callback,
    "providerId="+providerid+
    "&patientId="+patientId+
    "&encYear="+currentencyear
);
```

Meaning

Measures are loaded from

```
MeasureURL.Action
```

---

## Step 8

Studied Action Mapping.

```
MeasureURL.Action
=
com.glenwood.hcare.actions.chart.leafs.MeasureUrl
```

---

## Step 9

Reviewed MeasureUrl.java

Flow

```
Measurecode1.jsp
        ↓
MeasureURL.Action
        ↓
MeasureUrl.java
        ↓
CustomURLConnection
        ↓
GlaceRulesController/getPQRSMeasure
```

---

## Step 10

Observed important logic

If HEDIS configured

```
PQRSServices/getPQRSMeasuresInfo
```

Else

```
GlaceRulesController/getPQRSMeasure
```

Since

```sql
SELECT *
FROM hedis_configuration
WHERE hedis_configuration_reporting_year=2026;
```

returned

```
0 rows
```

Execution entered

```
GlaceRulesController/getPQRSMeasure
```

---

## Step 11

Verified provider configuration.

```sql
SELECT *
FROM macra_provider_configuration
WHERE macra_provider_configuration_provider_id=8
AND macra_provider_configuration_reporting_year=2026;
```

Result

```
Reporting Method = 2
Report Type = 2
```

---

## Step 12

Compared all providers.

Query

```sql
SELECT
e.emp_profile_fullname,
m.macra_provider_configuration_reporting_method
FROM macra_provider_configuration m
JOIN emp_profile e
ON e.emp_profile_empid =
m.macra_provider_configuration_provider_id
WHERE
m.macra_provider_configuration_reporting_year=2026;
```

Result

| Provider            | Reporting Method |
| ------------------- | ---------------- |
| Amy Dougherty       | 1                |
| Britain Wetzel      | 1                |
| Edward Farrell      | 1                |
| Melissa Shinder     | 1                |
| Michael Miller      | 1                |
| Robert Norton       | 1                |
| Test Doctor         | 1                |
| **Robert Filoramo** | **2**            |

Observation

Robert Filoramo is the **only provider** configured differently.

---

# Root Cause Analysis

## Confirmed Findings

Database configuration exists.

Encounter uses correct provider.

Provider mappings exist.

HEDIS disabled.

Backend API

```
GlaceRulesController/getPQRSMeasure
```

returns an empty response (inferred from UI behavior).

---

## Strong Suspected Root Cause

Provider

```
Robert Filoramo DPM
```

is configured with

```
Reporting Method = 2
```

while all working providers use

```
Reporting Method = 1
```

The backend API likely filters PQRS measures based on reporting method or related business rules, causing it to return no measures for this provider.

> **Status:** This is a strong correlation identified during investigation. The exact filtering logic inside `GlaceRulesController/getPQRSMeasure` was not reviewed, so this remains a probable root cause rather than a confirmed code defect.

---

# Detailed Technical Findings

## UI Decision

```
Measurecode1.jsp
```

If response contains JSON

```
Draw Measures
```

Else

```
Show

Measures not configured
```

---

## AJAX Call

```
MeasureURL.Action
```

Parameters

```
providerId
patientId
encYear
```

---

## Action

```
MeasureUrl.java
```

Calls

```
GlaceRulesController/getPQRSMeasure
```

using

```
CustomURLConnection
```

---

## Backend

Expected

```
Configured Measure JSON
```

Actual

Appears to return

```
Empty
```

---

# SQL Analysis and Scripts

## Investigation Queries

### Verify Provider Mapping

```sql
SELECT *
FROM quality_measures_provider_mapping
WHERE quality_measures_provider_mapping_provider_id = 8
AND quality_measures_provider_mapping_reporting_year = 2026;
```

Purpose

Verify provider configuration.

---

### Verify All Providers

```sql
SELECT DISTINCT
quality_measures_provider_mapping_provider_id,
quality_measures_provider_mapping_reporting_year
FROM quality_measures_provider_mapping
WHERE quality_measures_provider_mapping_reporting_year = 2026;
```

Purpose

Validate reporting-year configuration.

---

### Count Configurations

```sql
SELECT COUNT(*)
FROM quality_measures_provider_mapping
WHERE quality_measures_provider_mapping_reporting_year = 2026;
```

Purpose

Verify overall configuration volume.

---

### Compare Previous Years

```sql
SELECT *
FROM quality_measures_provider_mapping
WHERE quality_measures_provider_mapping_provider_id = 8
ORDER BY quality_measures_provider_mapping_reporting_year DESC;
```

Purpose

Compare yearly mappings.

---

### Verify Encounter

```sql
SELECT
encounter_id,
encounter_chartid,
encounter_service_doctor,
encounter_date
FROM encounter
WHERE encounter_id = 68961;
```

Purpose

Confirm service doctor.

---

### Verify HEDIS

```sql
SELECT *
FROM hedis_configuration
WHERE hedis_configuration_reporting_year = 2026;
```

Purpose

Determine execution path.

---

### Verify Provider Configuration

```sql
SELECT *
FROM macra_provider_configuration
WHERE macra_provider_configuration_provider_id = 8
AND macra_provider_configuration_reporting_year = 2026;
```

Purpose

Verify reporting method.

---

### Compare Provider Reporting Methods

```sql
SELECT
    e.emp_profile_fullname AS provider_name,
    m.macra_provider_configuration_provider_id,
    m.macra_provider_configuration_reporting_method,
    m.macra_provider_configuration_report_type
FROM macra_provider_configuration m
JOIN emp_profile e
    ON e.emp_profile_empid = m.macra_provider_configuration_provider_id
WHERE m.macra_provider_configuration_reporting_year = 2026
ORDER BY e.emp_profile_fullname;
```

Purpose

Identify configuration differences across providers.

---

# Code Investigation

## Files Reviewed

```
Measurecode1.jsp
```

```
ActionMappings.properties
```

```
MeasureUrl.java
```

---

## Important Flow

```
Measurecode1.jsp

↓

MeasureURL.Action

↓

MeasureUrl.performAction()

↓

CustomURLConnection.perform()

↓

GlaceRulesController/getPQRSMeasure

↓

JSON

↓

UI
```

---

# Fixes and Workarounds

## Temporary Investigation

* Verified provider mappings.
* Verified encounter provider.
* Verified reporting year.
* Verified provider configuration.

## Permanent Fix

**Not implemented during this investigation.**

The backend API `GlaceRulesController/getPQRSMeasure` should be reviewed to determine why it returns an empty response for Provider ID 8 when the provider is configured with Reporting Method = 2.

If business rules require PQRS only for Claim/Registry providers, the provider configuration may need correction after confirming expected reporting behavior with the customer.

---

# Validation and Testing

Completed

* Provider mapping validation.
* Reporting year validation.
* Encounter validation.
* Reporting method comparison.
* HEDIS validation.
* UI code review.
* Action layer review.

Pending

* Debug `GlaceRulesController/getPQRSMeasure`.
* Inspect returned JSON.
* Confirm reporting method business rules.
* Validate behavior after configuration/code changes.

---

# Risks and Side Effects

* Changing `macra_provider_configuration_reporting_method` without business confirmation could affect reporting workflows.
* Altering backend filters may impact all PQRS users.
* Ensure compatibility with Claim, Registry, EHR, and HEDIS reporting paths.

---

# Pending Work

1. Review `GlaceRulesController/getPQRSMeasure`.
2. Add logging for:

   * Provider ID
   * Patient ID
   * Reporting Year
   * Returned measure count
3. Verify whether Reporting Method = 2 should exclude PQRS.
4. Confirm expected behavior with the customer or product owner.
5. If needed, correct provider configuration or backend filtering logic.

---

# Lessons Learned

* Do not assume the "Measures not configured" message indicates missing database records; it may simply reflect an empty backend response.
* Validate configuration tables (`quality_measures_provider_mapping`, `macra_provider_configuration`, `hedis_configuration`) before investigating UI issues.
* Trace the complete request flow (JSP → Action → Backend API) to identify where data is lost.
* Compare configurations against working providers to identify subtle differences, such as reporting method.
* Add diagnostic logging in backend services to capture request parameters and returned measure counts, reducing future troubleshooting time.
