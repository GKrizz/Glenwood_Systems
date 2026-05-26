
# Day 355 – Dr Javvaji – MIPS Dashboard JSON Empty Issue

**Provider:** Dr. Javvaji
**Account:** njavvaji
**Module:** MIPS Dashboard / QPP Performance
**Issue Type:** Empty JSON / Dashboard Failure
**Application:** GlaceEMR

---

# Issue Summary

Dr. Javvaji was unable to view MIPS dashboard figures.

The following request returned incomplete/empty JSON:

```text id="xqzjkl"
https://njavvaji.glaceemr.com/Glace/jsp/Measures/MIPSPerformanceReport.Action?provider=1&year=2026&GlaceAjaxRequest=true
```

Returned payload:

```json id="6vjlwm"
{
  "macraProviderConfigurationReportingEnd": 1798693200000,
  "modifiedStage3": "f",
  "macraProviderConfigurationMailId": "",
  "macraProviderConfigurationReportingYear": 2026,
  "qualityMeasuresProviderMappingMeasureId": null,
  "IAMeasuresPoints": null,
  "iameasuresPoints": null,
  "macraProviderConfigurationReportingStart": 1767243600000,
  "macraProviderConfigurationReportType": 2,
  "macraProviderConfigurationACIStart": 1767243600000,
  "qualityMeasuresProviderMappingTitle": null,
  "macraProviderConfigurationACIEnd": 1798693200000,
  "qualityMeasuresProviderMappingPriority": null,
  "macraProviderConfigurationHardShipExemption": false,
  "iameasuresStatus": null,
  "macraProviderConfigurationReportingMethod": 2,
  "IAMeasuresStatus": null
}
```

The dashboard details API also failed:

```text id="w1rm6g"
MIPSPerformanceReport.Action?mode=10
```

Browser error:

```text id="r7c2uv"
SyntaxError: JSON.parse:
unexpected end of data
```

---

# Initial Observation

The dashboard configuration existed correctly:

* Reporting year = 2026
* Reporting method = EHR
* Provider configuration available

But:

```text id="fg9mpn"
Dashboard API raw response :: []
```

indicated backend APIs were returning empty data.

---

# Debugging Performed

## Added Debug Logs

### MIPSPerformanceReportAction.java

```java id="drj5x1"
System.out.println(
    "Dashboard API raw response :: ["
    + emrResponseBean.getData()
    + "]"
);

System.out.println(
    "Final dashboard response :: "
    + response
);

System.out.println(
    "Final dashboard data :: "
    + response.getData()
);
```

Result:

```text id="xvy2oc"
Dashboard API raw response :: []
```

---

# Exception Handling Added

## QPPPerformanceController.java

```java id="n32f8e"
catch(Exception e){
    GlaceLogger.LogException(e);
    e.printStackTrace();
    throw e;
}
```

---

# Data Gateway Validation

## Eligibility API Test

```bash id="v8v2db"
curl -v https://datagateway.glaceemr.com/DataGateway/MIPSEligibility/checkEligibility?npi=1649209655&year=2026
```

Response:

```json id="3bn9ef"
{
  "year":"2026",
  "message":"error",
  "required":"",
  "url":"https://qpp.cms.gov/participation-lookup?npi=1649209655&py=2026"
}
```

Observation:

* Eligibility API reachable
* Not primary root cause
* Dashboard still failing internally

---

# Dashboard API Investigation

## Actual Backend Request

```text id="r3s1cz"
POST QPPPerformance/getDashBoardDetails
```

## Query Params

```text id="9wtf1r"
isTrans=false
isByNpi=true
accountId=njavvaji
providerId=1
reportingYear=2026
```

## Request Body

```json id="vq0b0f"
{
  "excludeHospitalVisit": false,
  "excludeERVisit": false,
  "excludeNursingHomeVisit": false,
  "excludeAssistedLivingVisit": false
}
```

---

# Backend API Failure

## Direct API Call

```text id="zwhdqy"
http://172.18.24.92/glaceemr_backend_stable_v2/njavvaji/api/desktop/user/QPPPerformance/getDashBoardDetails
```

Returned:

```json id="4gnk0m"
{
  "success": false,
  "data": null,
  "errorMessage":
  "Transaction silently rolled back because it has been marked as rollback-only"
}
```

---

# Root Cause Investigation

Several configuration tables were validated successfully:

* `mips_weightage`
* `macra_configuration`
* `macra_provider_configuration`
* `quality_measures_provider_mapping`
* `macra_measures_rate`

All appeared populated correctly.

However, investigation identified a missing gateway configuration.

---

# Critical Finding

## Missing Gateway Entry

Query:

```sql id="9lvqq3"
SELECT *
FROM gateway_controller
WHERE gateway_controller_module_name =
      'icd10 for eCQM Services';
```

### Other Databases

| Database | Result  |
| -------- | ------- |
| dbc      | Present |
| calvary  | Present |
| mim      | Present |
| njavvaji | Missing |

---

# Existing Working Configuration

```text id="py1w3t"
gateway_controller_module_name :
icd10 for eCQM Services
```

```text id="pb4dzt"
gateway_controller_url :
https://datagateway.glaceemr.com/DataGatewayMediSpan/eCQMServices
```

---

# Root Cause

The `gateway_controller` entry required for:

```text id="2r8y5q"
icd10 for eCQM Services
```

was missing in the `njavvaji` database.

Because of this:

* Dashboard service could not call eCQM APIs
* Internal transaction failed
* Spring transaction marked rollback-only
* API returned empty JSON
* Frontend JSON.parse failed

---

# Fix Applied

## Insert Missing Gateway Configuration

```sql id="q4z7l8"
INSERT INTO gateway_controller(
    gateway_controller_id,
    gateway_controller_module_name,
    gateway_controller_url
)
VALUES(
    12,
    'icd10 for eCQM Services',
    'https://datagateway.glaceemr.com/DataGatewayMediSpan/eCQMServices'
);
```

---

# Validation

## Recheck Entry

```sql id="5xqsl9"
SELECT *
FROM gateway_controller
WHERE gateway_controller_module_name =
      'icd10 for eCQM Services';
```

Expected:

| gateway_controller_id | module_name             | url     |
| --------------------- | ----------------------- | ------- |
| 12                    | icd10 for eCQM Services | Present |

---

# Additional Validation Queries

## Verify MIPS Weightage

```sql id="5q7j0x"
SELECT
    mips_weightage_quality AS qualityWt,
    mips_weightage_pi AS piWt,
    mips_weightage_ia AS iaWt,
    mips_weightage_cost AS cost
FROM mips_weightage
WHERE mips_weightage_year = 2026;
```

---

## Verify MACRA Configuration

```sql id="x9m6v0"
SELECT *
FROM macra_configuration
WHERE macra_configuration_year = 2026;
```

---

## Verify Provider Configuration

```sql id="tm4g1d"
SELECT
    macra_provider_configuration_provider_id,
    macra_provider_configuration_reporting_year,
    macra_provider_configuration_reporting_method
FROM macra_provider_configuration
WHERE macra_provider_configuration_reporting_year = 2026
  AND macra_provider_configuration_provider_id = 1;
```

---

# Additional Findings

## Reporting Method

```sql id="0w5mya"
SELECT
CASE
 WHEN macra_provider_configuration_reporting_method=1
      THEN 'Claims'
 WHEN macra_provider_configuration_reporting_method=2
      THEN 'EHR'
 WHEN macra_provider_configuration_reporting_method=3
      THEN 'Registry/QCDR'
END
FROM macra_provider_configuration
WHERE macra_provider_configuration_provider_id=1
AND macra_provider_configuration_reporting_year=2026;
```

Result:

```text id="d8qvwe"
EHR
```

---

# Data Validation

## Measures Present

Validated:

* Measure mappings
* Denominator/Numerator counts
* NPI mappings
* CMS IDs
* Patient entries

No major data issues identified.

---

# Cleanup Queries

## Delete Entry if Needed

```sql id="8y3z3m"
DELETE FROM gateway_controller
WHERE gateway_controller_id = 12;
```

or

```sql id="s3c9a7"
DELETE FROM gateway_controller
WHERE gateway_controller_module_name =
      'icd10 for eCQM Services';
```

---

# Final Root Cause

The MIPS dashboard failed because:

```text id="0mdz7w"
gateway_controller entry for
"icd10 for eCQM Services"
was missing
```

This caused:

* Backend transaction rollback
* Empty dashboard API response
* JSON parsing failure on frontend
* MIPS dashboard data not loading

---

# Resolution

After restoring the missing gateway configuration:

* Dashboard APIs started resolving correctly
* Backend transaction failures stopped
* JSON payload populated successfully
* Dr. Javvaji could view MIPS figures again

---

# Final Outcome

| Component                 | Status   |
| ------------------------- | -------- |
| MIPS Dashboard            | Fixed    |
| Dashboard API             | Working  |
| JSON Parse Error          | Resolved |
| Backend Rollback          | Resolved |
| Gateway Controller Config | Restored |
| MIPS Figures              | Visible  |
