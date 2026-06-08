
# MIPS 2026 Monthly Report JAR Execution, Debugging, RCA, and Deployment Documentation

## Overview

This document captures the complete investigation, debugging, validation, and deployment process for the **MIPS 2026 Monthly Report JAR execution** used in the GlaceEMR ecosystem.

The work involved:

* Reverse engineering an existing JAR
* Updating reporting year logic
* Rebuilding the JAR
* Debugging backend API failures
* Identifying missing MIPS calculation data
* Validating PDF generation
* Fixing Java runtime compatibility
* Testing email delivery
* Deploying the final JAR for all accounts

This document serves as:

* Technical implementation reference
* Troubleshooting guide
* Root Cause Analysis (RCA)
* Operational deployment guide
* Knowledge transfer documentation

---

# Background / Context

The organization maintains a Java-based batch process that:

1. Reads all MIPS-enabled accounts from SSO
2. Calls backend Spring APIs
3. Generates MIPS monthly reports
4. Generates HTML/PDF reports
5. Emails reports to configured providers

The existing JAR:

* was already present on SFTP
* was reverse engineered (decompiled)
* updated to support reporting year `2026`
* rebuilt and redeployed

The goal was:

* validate MIPS monthly report generation for 2026
* ensure PDF/email generation works
* finally execute reports for all accounts

---

# Problem Statement

## Initial Failure

Running the updated JAR resulted in:

```text
java.net.ConnectException: Connection timed out
```

Later:

```text
java.io.IOException: Server returned HTTP response code: 500
```

And backend logs showed:

```text
org.springframework.transaction.UnexpectedRollbackException:
Transaction silently rolled back because it has been marked as rollback-only
```

## Symptoms

* API reachable
* Backend server reachable
* No PDF generated
* No HTML generated
* No email received
* Transaction rollback occurred
* `macra_measures_rate` table had no data for 2026 in `d2desktop`

---

# Impact

## Technical Impact

* Monthly MIPS reports could not be generated
* PDF/email pipeline failed
* Scheduled reporting process blocked

## Business Impact

* Providers would not receive monthly MIPS performance reports
* Reporting automation for 2026 could not proceed
* Operational reporting validation delayed

---

# Environment Details

| Component     | Value                         |
| ------------- | ----------------------------- |
| Application   | GlaceEMR                      |
| Module        | MIPS Monthly Report           |
| Language      | Java                          |
| Runtime       | Java 10                       |
| Backend       | Spring Boot                   |
| Web Server    | Tomcat 10                     |
| DB            | PostgreSQL                    |
| PDF Generator | wkhtmltopdf                   |
| Environment   | dev2 / MID                    |
| SFTP          | ftp.glaceemr.com              |
| Shared Path   | `/mnt/vs22shared/mid/tmp/mu/` |

---

# System Components Involved

## APIs

### Main API

```http
POST /api/emr/glacemonitor/mipsperformance/sendMIPSPDFviaEmail
```

### SSO APIs

```http
https://sso.glaceemr.com/SearchAccounts?mips=1
```

```http
https://sso.glaceemr.com/TestSSOAccess?accountId=<account>
```

---

# Investigation Timeline

---

## Phase 1 — JAR Execution Failure

### Initial Attempt

Executed:

```bash
java MIPS
```

Observed:

```text
Connection timed out
```

---

## Phase 2 — API Reachability Validation

Validated backend availability:

```bash
wget "http://172.18.24.162/glaceemr_backend_stage/"
```

Result:

```text
HTTP 200
```

Conclusion:

* backend reachable
* issue inside API execution

---

## Phase 3 — Direct API Testing

Executed CURL:

```bash
curl -X POST \
"http://172.18.24.162:8080/glaceemr_backend_stage/api/emr/glacemonitor/mipsperformance/sendMIPSPDFviaEmail?reportingYear=2026&accountID=d2desktop&dbname=d2desktop"
```

Response:

```json
{
  "success": false,
  "errorMessage": "Transaction silently rolled back because it has been marked as rollback-only"
}
```

---

## Phase 4 — Backend Log Analysis

Analyzed:

```bash
tail -f catalina.out
```

Observed:

```text
UnexpectedRollbackException
```

Additional logs:

* NullPointerExceptions
* FileNotFoundException
* Unauthorized exceptions
* Validator DTD missing

However, these were unrelated background issues.

---

## Phase 5 — wkhtmltopdf Validation

Validated:

```bash
which wkhtmltopdf
wkhtmltopdf --version
```

Result:

```text
wkhtmltopdf 0.12.6.1
```

Conclusion:

* PDF engine installed correctly

---

## Phase 6 — Database Investigation

### Critical Discovery

Query:

```sql
SELECT COUNT(*)
FROM macra_measures_rate
WHERE macra_measures_rate_reporting_year = 2026;
```

Result:

```text
0
```

This became the actual root cause.

---

## Phase 7 — Alternate Account Testing

Selected `mid` account.

Validation showed:

```sql
SELECT COUNT(*)
FROM macra_measures_rate
WHERE macra_measures_rate_reporting_year = 2026;
```

Result:

```text
27
```

This confirmed:

* API logic worked
* backend logic worked
* issue was data-specific

---

## Phase 8 — Email Configuration

Updated provider emails:

```sql
UPDATE macra_provider_configuration
SET macra_provider_configuration_mailid = 'gobalakrishnan@glenwoodsystems.com'
WHERE macra_provider_configuration_provider_id IN (1,11)
AND macra_provider_configuration_reporting_year = 2026;
```

---

## Phase 9 — Successful End-to-End Validation

Generated successfully:

* HTML
* PDF
* Email delivery

Files created:

```text
/mnt/vs22shared/mid/tmp/mu/
```

Generated:

* SolomonBeraki.pdf
* TestDoctor.pdf

Mail successfully received.

---

## Phase 10 — Java Compatibility Failure

Server execution failed:

```text
UnsupportedClassVersionError
```

Cause:

* JAR compiled using Java 17
* server runtime was Java 10

---

## Phase 11 — Java Compatibility Fix

Recompiled using:

```bash
javac --release 10 MIPS.java
```

Validated:

```bash
javap -verbose MIPS.class | grep "major version"
```

Result:

```text
major version: 54
```

---

## Phase 12 — Final Deployment

Uploaded updated JAR to:

```text
/Temp/Gobal/MIPS_2026/
```

Backups maintained.

---

# Root Cause Analysis

## Primary Root Cause

### Missing Data in `macra_measures_rate`

The account `d2desktop` contained:

```sql
SELECT COUNT(*)
FROM macra_measures_rate
WHERE macra_measures_rate_reporting_year = 2026;
```

Result:

```text
0
```

The MIPS report generation process depends on:

* provider configuration
* patient entries
* measure mappings
* calculated measure rates

Since `macra_measures_rate` had no data:

* report generation failed
* HTML/PDF generation never started
* transaction rolled back

---

## Secondary Root Cause

### Java Runtime Version Mismatch

JAR compiled using newer JDK.

Server runtime:

```text
Java 10
```

Compiled class version:

```text
61.0
```

Supported runtime:

```text
54.0
```

Fix:

* compile using `--release 10`

---

# Detailed Technical Findings

---

# APIs

## Monthly Report API

```http
POST /api/emr/glacemonitor/mipsperformance/sendMIPSPDFviaEmail
```

Parameters:

* reportingYear
* accountID
* dbname

Payload:

```json
{
  "excludeHospitalVisit": false,
  "excludeERVisit": false,
  "excludeNursingHomeVisit": false
}
```

---

# Database Tables

| Table                             | Purpose                                |
| --------------------------------- | -------------------------------------- |
| macra_measures_rate               | Final calculated MIPS performance data |
| macra_provider_configuration      | Provider email/report configuration    |
| quality_measures_provider_mapping | Provider measure mappings              |
| quality_measures_patient_entries  | Patient-level MIPS entries             |
| mips_scores                       | Provider MIPS scores                   |
| mips_notification                 | Threshold configuration                |

---

# SQL Analysis and Scripts

## Investigation Queries

### Check measure rates

```sql
SELECT COUNT(*)
FROM macra_measures_rate
WHERE macra_measures_rate_reporting_year = 2026;
```

Purpose:

* validate existence of generated MIPS data

---

### Provider-wise measure count

```sql
SELECT macra_measures_rate_provider_id,
       COUNT(*) AS total_records
FROM macra_measures_rate
WHERE macra_measures_rate_reporting_year = 2026
GROUP BY macra_measures_rate_provider_id;
```

Purpose:

* verify provider-level calculation presence

---

### Validate patient entries

```sql
SELECT COUNT(*)
FROM quality_measures_patient_entries
WHERE quality_measures_patient_entries_reporting_year = 2026;
```

Purpose:

* verify raw patient measure data exists

---

### Validate provider configuration

```sql
SELECT macra_provider_configuration_provider_id,
       macra_provider_configuration_mailid
FROM macra_provider_configuration
WHERE macra_provider_configuration_reporting_year = 2026;
```

Purpose:

* verify email configuration

---

## Fix Scripts

### Configure temporary email

```sql
UPDATE macra_provider_configuration
SET macra_provider_configuration_mailid = 'gobalakrishnan@glenwoodsystems.com'
WHERE macra_provider_configuration_provider_id IN (1,11)
AND macra_provider_configuration_reporting_year = 2026;
```

Purpose:

* redirect reports for testing

Impact:

* only affects email delivery destination

Rollback:

```sql
UPDATE macra_provider_configuration
SET macra_provider_configuration_mailid = ''
WHERE macra_provider_configuration_provider_id IN (1,11)
AND macra_provider_configuration_reporting_year = 2026;
```

---

## Backup Script

```sql
\copy (
SELECT *
FROM macra_provider_configuration
WHERE macra_provider_configuration_reporting_year = 2026
)
TO 'macra_provider_configuration_backup.csv'
WITH CSV HEADER;
```

Purpose:

* backup before update

---

# Code Changes

## File Modified

```text
MIPS.java
```

---

## Original Behavior

Hardcoded:

```java
String accountID = "mid";
```

Only processed single account.

---

## Updated Behavior

Changed to:

```java
String accountID = (String)accountsDetails.get("accountid");
```

Now processes:

* all MIPS-enabled accounts from SSO

---

## Loop Logic Fix

Original:

* used `boolean isdemo`

Updated:

```java
while (i < accountsList.length())
```

Cleaner iteration across accounts.

---

# Build and Packaging

## Compile

```bash
javac --release 10 MIPS.java
```

---

## Verify

```bash
javap -verbose MIPS.class | grep "major version"
```

Expected:

```text
major version: 54
```

---

## Create JAR

```bash
jar cvfm MIPS2026.jar manifest.txt MIPS.class org/json/*.class
```

---

# Deployment Steps

## Upload to SFTP

```bash
sftp -oPort=8444 glenwood@ftp.glaceemr.com
```

Path:

```text
/Temp/Gobal/MIPS_2026/
```

---

## Backup Existing JAR

```bash
rename MIPS2026.jar MIPS2026_previous_backup.jar
```

---

## Upload Latest

```bash
put MIPS2026.jar
```

---

# Validation and Testing

## API Validation

Validated:

* backend reachable
* API callable
* response structure

---

## Database Validation

Verified:

* provider mappings
* patient entries
* measure rates
* thresholds
* mail configuration

---

## PDF Validation

Generated:

* HTML
* PDF

Verified in:

```text
/mnt/vs22shared/mid/tmp/mu/
```

---

## Email Validation

Received:

* MIPS report mail
* PDF attachment

---

# Risks and Side Effects

## Risks

### Running Across All Accounts

Potential issues:

* accounts with missing measure data
* accounts without mail configuration
* inactive providers
* transaction rollback per account

---

## Performance Considerations

Processing all accounts:

* sequential API calls
* possible long runtime
* high DB load

---

## Operational Risks

* mass email generation
* PDF generation load
* shared storage growth

---

# Pending Work

## Recommended Follow-ups

* identify accounts missing `macra_measures_rate`
* improve logging inside backend API
* add explicit validation before PDF generation
* skip invalid accounts gracefully
* generate execution summary report

---

# Lessons Learned

## Debugging Insights

* API reachability ≠ business success
* transaction rollback often hides actual failure
* missing data caused entire report failure

---

## Prevention Recommendations

### Add Prechecks

Before PDF generation:

* validate measure rates exist
* validate mail configuration
* validate provider mappings

---

## Logging Improvements

Current backend logs:

* insufficient root cause visibility

Recommended:

* explicit error logs for missing measure data
* account-wise execution logging

---

## Build Process Improvements

Always compile using target runtime:

```bash
javac --release 10
```

Avoid runtime incompatibility.

---

# Final Outcome

## Successfully Achieved

✅ Decompiled and modified existing JAR
✅ Updated reporting year logic
✅ Identified missing data issue
✅ Validated backend/API functionality
✅ Fixed Java compatibility issue
✅ Generated PDFs successfully
✅ Received email reports successfully
✅ Rebuilt deployable JAR
✅ Uploaded production-ready JAR to SFTP
✅ Prepared JAR for all-account execution

---

# Final Deployment Artifact

## Active JAR

```text
/Temp/Gobal/MIPS_2026/MIPS2026.jar
```

---

# Backup Artifacts

```text
MIPS2026_old_jun_03_2026.jar
MIPS2026_java17_backup.jar
MIPS2026_previous_backup.jar
```
