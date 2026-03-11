Below is a **clean, professional, full README.md** combining everything you wrote, but organized the way **engineering teams typically document production bugs and fixes**.
You can **paste this directly into GitHub / GitLab / Confluence**.

---

# MACRA Tab Not Loading – Debug & Resolution Guide

**Issue ID:** MACRA-TAB-LOAD-FAILURE
**Reported By:** Dr. Bethala
**Date Reported:** March 10, 2026
**Environment:** Production
**Reporting Year:** 2026

---

# Issue Summary

The **MACRA tab failed to load** when providers attempted to complete measures before signing charts.

Users observed:

* Blank page
* Red ❌ error icon
* Message indicating failure to retrieve ECQM measure status

This prevented providers from reviewing and completing **MIPS quality measures prior to chart sign-off**.

---

# Impact

Providers were unable to:

* Open the **MACRA flowsheet**
* Review **unmet measures**
* Complete required **MIPS documentation**

This impacted the following provider:

| Field          | Value      |
| -------------- | ---------- |
| Provider       | Dr Bethala |
| Provider ID    | 7          |
| Reporting Year | 2026       |

---

# User Report

> We are unable to open the Macra tab upon completion of the chart.
> We either get a blank page or a big red X mark on the page.
> This prevents us from completing measures that have not been met before signing the chart.

---

# MACRA Tab URL

```
https://glace-gwt.glaceemr.com/sdesktop/glaceemr.html
?macraIntegration=true
&patientId=25268
&chartId=25275
&encounterId=73667
&providerId=7
&encdate=2026-03-10
&userId=1
&userName=Demodoctor
&contextPath=vbethala
```

---

# API Call Triggered

The MACRA tab loads measures using the following API.

### Endpoint

```
GET /api/desktop/user/QPPPerformance/getCQMStatusByPatient
```

### Full API Request

```
https://glace-gwt.glaceemr.com/glaceemr_backend_stable/vbethala/api/desktop/user/QPPPerformance/getCQMStatusByPatient
```

### Query Parameters

| Parameter  | Value    |
| ---------- | -------- |
| patientID  | 25268    |
| providerId | 7        |
| accountId  | vbethala |
| year       | 2026     |
| userId     | 7        |
| dbname     | vbethala |

---

# API Response

### Error Response

```json
{
  "login": null,
  "success": false,
  "data": null,
  "errorMessage": "Cannot invoke \"String.split(String)\" because the return value of \"MacraProviderQDM.getMeasures()\" is null"
}
```

### UI Error

```
❌ Unable to get ECQM Measure status
```

---

# Root Cause

The backend method **saveOrUpdatePatientQDMStatus()** executes the following line:

```java
String[] measureIds = macraProviderQDM.getMeasures().split(",");
```

However:

```
macraProviderQDM.getMeasures() = NULL
```

Which results in:

```
NullPointerException
```

Therefore the API fails and the MACRA tab does not load.

---

# Code Flow

### Step 1 – Determine Reporting Mode

The system checks whether reporting is **Individual or Group**.

```java
Boolean isIndividual = measureService.checkGroupOrIndividual(year);
```

Method:

```java
public Boolean checkGroupOrIndividual(int year)
```

---

### Reporting Mode Logic

| macra_configuration_type | Mode                 |
| ------------------------ | -------------------- |
| 0                        | Individual Reporting |
| 1                        | Group Reporting      |

---

### Step 2 – Provider Override

When **Group Reporting** is enabled:

```
providerId = -1
userId = -1
```

Meaning:

The system loads **group configuration instead of provider configuration**.

---

# Database Investigation

### MACRA Configuration

```sql
SELECT *
FROM macra_configuration
WHERE macra_configuration_year = 2026;
```

Result:

```
macra_configuration_type = 1
```

Meaning:

```
Group Reporting Enabled
```

---

### MACRA Provider Configuration

```sql
SELECT *
FROM macra_provider_configuration
WHERE macra_provider_configuration_provider_id = -1
AND macra_provider_configuration_reporting_year = 2026;
```

Result:

```
Record Exists
```

---

### Quality Measure Mapping

```sql
SELECT *
FROM quality_measures_provider_mapping
WHERE quality_measures_provider_mapping_provider_id = -1
AND quality_measures_provider_mapping_reporting_year = 2026;
```

Result:

```
0 rows
```

---

# Why the Crash Happened

Backend query:

```sql
SELECT string_agg(quality_measures_provider_mapping_measure_id, ',')
```

Because no rows existed:

```
string_agg → NULL
```

Java then executed:

```java
null.split(",")
```

Which caused:

```
NullPointerException
```

Result:

```
MACRA tab failed to load
```

---

# Database Queries Used for Investigation

Provider configuration check:

```sql
SELECT *
FROM macra_provider_configuration
WHERE macra_provider_configuration_provider_id = 7
AND macra_provider_configuration_reporting_year = 2026;
```

Measure mapping:

```sql
SELECT *
FROM quality_measures_provider_mapping
WHERE quality_measures_provider_mapping_provider_id = 7
AND quality_measures_provider_mapping_reporting_year = 2026;
```

Measure aggregation used by backend:

```sql
SELECT string_agg(quality_measures_provider_mapping_measure_id, ',')
FROM quality_measures_provider_mapping
WHERE quality_measures_provider_mapping_provider_id = 7
AND quality_measures_provider_mapping_reporting_year = 2026
AND quality_measures_provider_mapping_measure_id NOT LIKE 'IA_%';
```

---

# Production Data Export

Exported provider data from production.

```sql
\copy (
SELECT *
FROM macra_provider_configuration
WHERE macra_provider_configuration_provider_id = 7
AND macra_provider_configuration_reporting_year = 2026
)
TO '/tmp/macra_provider_configuration_p7_2026.csv'
WITH CSV HEADER;
```

```sql
\copy (
SELECT *
FROM quality_measures_provider_mapping
WHERE quality_measures_provider_mapping_provider_id = 7
AND quality_measures_provider_mapping_reporting_year = 2026
)
TO '/tmp/quality_measures_provider_mapping_p7_2026.csv'
WITH CSV HEADER;
```

Generated files:

```
/tmp/macra_provider_configuration_p7_2026.csv
/tmp/quality_measures_provider_mapping_p7_2026.csv
```

---

# File Transfer

Files uploaded to internal FTP.

```
ftp.glaceemr.com
/Temp/Gobal/
```

Files:

```
macra_provider_configuration_p7_2026.csv
quality_measures_provider_mapping_p7_2026.csv
```

---

# Import Into Test Environment

Downloaded files and imported.

```sql
\copy quality_measures_provider_mapping
FROM 'quality_measures_provider_mapping_p7_2026.csv'
WITH CSV HEADER;
```

```sql
\copy macra_provider_configuration
FROM 'macra_provider_configuration_p7_2026.csv'
WITH CSV HEADER;
```

---

# Temporary Fix Applied

To restore functionality quickly:

```sql
UPDATE macra_configuration
SET macra_configuration_type = 0
WHERE macra_configuration_year = 2026;
```

This changed the system to:

```
Individual Reporting
```

System then used:

```
providerId = 7
```

Instead of:

```
providerId = -1
```

Since provider **7 had measures configured**, the MACRA tab loaded successfully.

---

# Verification

After applying the fix:

```sql
SELECT *
FROM macra_configuration
WHERE macra_configuration_year = 2026;
```

Result:

```
macra_configuration_type = 0
```

MACRA tab successfully loaded.

---

# Recommended Permanent Fix

Instead of disabling Group Reporting, measures should be configured for **provider -1**.

Example:

```sql

UPDATE macra_configuration SET macra_configuration_type = 0 WHERE macra_configuration_year = 2026;

SELECT * FROM macra_configuration WHERE macra_configuration_year = 2026;
```
```sql
SELECT * FROM macra_configuration WHERE macra_configuration_year = 2026;
```

---

# Prevention

Before enabling **Group Reporting**, verify:

1. `macra_provider_configuration` exists for **provider -1**
2. `quality_measures_provider_mapping` exists for **provider -1**
3. `string_agg()` query returns valid measures

Validation query:

```sql
SELECT string_agg(quality_measures_provider_mapping_measure_id, ',')
FROM quality_measures_provider_mapping
WHERE quality_measures_provider_mapping_provider_id = -1
AND quality_measures_provider_mapping_reporting_year = 2026;
```

---

# Key Learning

The issue occurred due to **configuration mismatch between reporting mode and measure mapping**.

System flow:

```
Group reporting enabled
↓
providerId switched to -1
↓
No measures configured for provider -1
↓
string_agg returned NULL
↓
split() caused NullPointerException
↓
MACRA tab failed to load
```

---

# Status

✔ Issue resolved
✔ MACRA tab loads successfully
✔ Measures displayed correctly

---
