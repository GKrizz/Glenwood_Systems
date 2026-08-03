
# Case #247663 

# CCDA Clinical Summary XML Not Generated for UHS Account

---

# Overview

This document captures the complete investigation, root cause analysis (RCA), implementation details, validation steps, and long-term maintenance notes for **Case #247663**.

The reported issue was that **CCDA Clinical Summary could not be downloaded as an XML file** for the **UHS** tenant. Initially, the customer believed the system was downloading an **XSL** file instead of XML. Investigation showed that the download mechanism was functioning correctly, but the **Clinical Summary XML was never generated** for specific encounters.

The issue was ultimately traced to a **missing database table** in the UHS tenant schema.

---

# Background / Context

## Business Requirement

The customer exports CCDA Clinical Summary files from GlaceEMR to import into an external Care Management Platform.

The external system requires:

* Clinical Summary XML
* CDA document
* PDF
* Stylesheet

The export process is initiated from:

```
Patient Chart
    → Utilities
        → Export Clinical Summary
```

---

## Customer Report

Customer stated:

* XML is not downloading.
* Download appears to be an XSL file.
* Their external system cannot import the downloaded content.

---

# Problem Statement

## Observed Behavior

For certain patients:

* Export workflow completed successfully.
* ZIP file was downloaded.
* ZIP contained:

```
Encounter.cda
PDF
Stylesheet
SHA512
```

But **Clinical-Summary.xml** was missing.

Example:

```
OUT-07-08-2026-02-05-00-671-Encounter-7866.cda

Missing:

OUT-07-08-2026-02-05-00-671-Encounter-7866-Clinical-Summary.xml
```

---

## Expected Behavior

ZIP should contain:

```
Encounter.cda
Encounter-Clinical-Summary.xml
PDF
Stylesheet
SHA512
```

---

# Impact

## Business Impact

Customer could not import patient data into their external Care Management platform.

Since the XML file was absent:

* CCDA exchange failed.
* Clinical interoperability was broken.
* Customer assumed download process was incorrect.

---

# Environment Details

| Item                 | Value              |
| -------------------- | ------------------ |
| Customer             | UHS                |
| Account No           | 006873             |
| Patient Example      | FORRES W. HAGGERTY |
| Encounter            | 7866               |
| Working Test Patient | KIRK TEST          |
| Encounter            | 3509               |

---

# System Components Involved

## UI

```
Patient Chart
    ↓
Utilities
    ↓
Export Clinical Summary
```

---

## Controller

```
CDADownloadOptionView.Action
```

Returns encounter list.

---

## Disclosure Screen

```
disclosureFrm.jsp
```

---

## CDA Generator

```
GenerateCDA.Action
```

Responsible for:

* Generate CDA
* Generate Clinical Summary XML
* Generate PDF
* Generate SHA512
* Copy Stylesheet
* Compress files

---

## Download Endpoint

```
ClaimDownload
```

Downloads generated ZIP.

---

## Spring Boot Endpoint

```
POST

/CDAGeneration/GenerateCda
```

---

## File System

```
/mnt1/vs13shared/uhs/log/CDAFiles/

├── cda_Outbox
├── CDAPDF
├── Compressed
├── Signed_cda
└── CDAInterface
```

---

# End-to-End Workflow

```text
Patient Chart
      │
      ▼
Utilities
      │
      ▼
Export Clinical Summary
      │
      ▼
CDADownloadOptionView.Action
      │
      ▼
Select Encounter
      │
      ▼
Download Encounter
      │
      ▼
Disclosure Form
      │
      ▼
Save
      │
      ▼
GenerateCDA.Action
      │
      ▼
Generate CDA
Generate XML
Generate PDF
Generate SHA512
Copy Stylesheet
Compress ZIP
      │
      ▼
ClaimDownload
      │
      ▼
ZIP Download
```

---

# Investigation Timeline

## Phase 1

Verified UI workflow.

Confirmed:

```
Utilities

↓

Export Clinical Summary
```

worked correctly.

---

## Phase 2

Captured network requests.

Verified:

```
CDADownloadOptionView.Action
```

returned encounter list successfully.

---

Verified:

```
Disclosure Form
```

opened correctly.

---

Verified:

```
GenerateCDA.Action
```

returned HTML containing download links.

---

Verified:

```
ClaimDownload
```

downloaded ZIP correctly.

Therefore:

UI flow was working.

---

## Phase 3

Compared generated files.

### Test Patient

```
Encounter 3509
```

Generated:

```
Encounter.cda

Encounter-Clinical-Summary.xml

ZIP
```

Everything correct.

---

### Production Patient

```
Encounter 7866
```

Generated:

```
Encounter.cda
```

Missing:

```
Encounter-Clinical-Summary.xml
```

---

## Phase 4

Verified server filesystem.

Checked:

```
cda_Outbox
```

For test patient:

```
Encounter.cda

Encounter-Clinical-Summary.xml
```

For failing patient:

```
Encounter.cda
```

Only CDA existed.

XML never created.

---

Verified ZIP directory.

Observed:

Working ZIP size:

```
~83 KB
```

Failing ZIP:

```
~36 KB
```

Confirming XML was absent before compression.

---

## Phase 5

Called Spring Boot endpoint directly.

Successful request:

```http
POST /CDAGeneration/GenerateCda
```

for Encounter 3509

Response:

```json
{
  "data": [
    "OUT-07-10-2026-10-06-02-287-Encounter-3509.cda"
  ]
}
```

Generated successfully.

---

Failed request:

Encounter 7866

Returned:

```json
{
  "success": false,
  "errorMessage": "Transaction silently rolled back because it has been marked as rollback-only"
}
```

---

## Phase 6

Reviewed server logs.

Used:

```bash
grep -nE "CDAGenerationController\.java|CDAGenerationServiceImpl\.java" *_EXCEPTION_2026-07-10.log

grep -n 'Transaction silently rolled back' *_EXCEPTION_2026-07-10.log
```

Confirmed rollback occurred during CDA generation.

---

## Phase 7

Investigated database.

Compared schema.

Local environment:

```sql
SELECT to_regclass('public.encounter_chart_app_reference_values');
```

Returned:

```
encounter_chart_app_reference_values
```

UHS database:

```sql
SELECT to_regclass('public.encounter_chart_app_reference_values');
```

Returned:

```
NULL
```

The table did not exist.

---

# Root Cause Analysis

## Root Cause

The tenant database **did not contain the table**:

```
encounter_chart_app_reference_values
```

During Clinical Summary XML generation, application logic attempted to access this table.

Since it did not exist:

* Transaction failed.
* Transaction marked rollback-only.
* XML generation aborted.
* CDA file had already been written before rollback.
* ZIP therefore contained only CDA.

---

## Contributing Factors

* Tenant schema mismatch.
* Missing migration.
* No startup validation.
* Partial file generation occurred before transaction failure.
* Error surfaced only as generic rollback exception.

---

## Misleading Observations

Initially suspected:

* Browser downloaded XSL.
* Download endpoint issue.
* ZIP compression issue.
* ClaimDownload issue.

All were incorrect.

Actual failure occurred during XML generation.

---

# Detailed Technical Findings

## APIs

### Encounter List

```
CDADownloadOptionView.Action
```

Purpose:

Returns encounters.

---

### Disclosure

```
disclosureFrm.jsp
```

Purpose:

Collect disclosure metadata.

---

### CDA Generation

```
GenerateCDA.Action
```

Responsibilities:

* Generate CDA
* Clinical Summary XML
* PDF
* SHA512
* ZIP

---

### Spring Endpoint

```
POST

/CDAGeneration/GenerateCda
```

Returned rollback error for failing encounter.

---

## Files Generated

Working:

```
Encounter.cda

Encounter-Clinical-Summary.xml

ZIP

SHA512

PDF
```

Failing:

```
Encounter.cda

ZIP

No XML
```

---

# SQL Analysis and Scripts

## Investigation Queries

### Check Encounter

```sql
SELECT *
FROM encounter
WHERE encounter_id = 7866;
```

Purpose:

Validate encounter data.

Type:

Investigation query.

---

### Provider Validation

```sql
SELECT
    encounter_id,
    encounter_ref_doctor,
    encounter_service_doctor,
    encounter_ass_provider,
    encounter_chartid
FROM encounter
WHERE encounter_id = 7866;
```

Purpose:

Validate provider references.

Type:

Investigation query.

---

### Referring Doctor Validation

```sql
SELECT
    e.encounter_id,
    e.encounter_ref_doctor,
    rd.referring_doctor_uniqueid
FROM encounter e
LEFT JOIN referring_doctor rd
ON rd.referring_doctor_uniqueid =
e.encounter_ref_doctor
WHERE e.encounter_id = 7866;
```

Purpose:

Validate foreign key references.

Type:

Investigation query.

---

### Verify Table Exists

```sql
SELECT to_regclass(
'public.encounter_chart_app_reference_values'
);
```

Purpose:

Check schema.

Type:

Investigation query.

---

### Check Mapping

```sql
SELECT *
FROM encounter_chart_app_reference_values
WHERE encounter_reason =
(
SELECT encounter_reason
FROM encounter
WHERE encounter_id = 7866
);
```

Purpose:

Verify mapping data.

Type:

Validation query.

---

### Migration Verification

```sql
SELECT *
FROM flyway_schema_history
WHERE script ILIKE
'%encounter_chart_app_reference_values%'
OR description ILIKE
'%encounter_chart_app_reference_values%';
```

Purpose:

Check migration history.

Type:

Investigation query.

---

## Fix Script

```sql
CREATE TABLE encounter_chart_app_reference_values
(
    encounter_reason INTEGER NOT NULL,
    app_reference_values_statusid INTEGER NOT NULL,

    CONSTRAINT encounter_chart_app_reference_values_pkey
    PRIMARY KEY
    (
        encounter_reason,
        app_reference_values_statusid
    )
);
```

Purpose:

Create missing lookup table.

Type:

Permanent schema fix.

### Affected Table

```
encounter_chart_app_reference_values
```

### Performance

Primary key index automatically created.

No performance concerns identified.

### Rollback

```sql
DROP TABLE encounter_chart_app_reference_values;
```

(Not recommended unless migration is reverted.)

---

# Code Changes

No application code modifications were discussed.

Issue was resolved by restoring the expected database schema.

---

# Fixes and Workarounds

## Temporary Workaround

None identified.

---

## Permanent Fix

Create the missing table:

```text
encounter_chart_app_reference_values
```

Once created:

* CDA generation succeeded.
* Clinical Summary XML generated.
* ZIP contained expected files.

---

## Deployment Considerations

Ensure all tenant databases receive the schema migration.

---

# Validation and Testing

Validation performed using:

## Test Patient

```
KIRK TEST
Encounter 3509
```

Verified:

* CDA
* XML
* ZIP
* PDF

---

## Production Patient

```
FORRES W. HAGGERTY
Encounter 7866
```

Before fix:

```
Only CDA
```

After creating table:

Clinical Summary generation worked successfully.

---

## Filesystem Validation

Checked:

```
cda_Outbox
```

Confirmed XML creation.

Checked:

```
Compressed
```

Confirmed ZIP included XML.

---

## API Validation

Verified:

```
POST /CDAGeneration/GenerateCda
```

Returned success after schema correction.

---

# Risks and Side Effects

## Production Risks

* Missing tenant migrations can cause runtime failures.
* Schema drift between tenants can introduce inconsistent behavior.

## Data Integrity Risks

Low.

The fix only restores an expected lookup table.

## Performance Risks

None identified.

## Backward Compatibility

No backward compatibility concerns were discussed.

---

# Pending Work

* Verify that all customer/tenant databases include `encounter_chart_app_reference_values`.
* Audit Flyway (or legacy migration) history to determine why the migration was absent for the UHS tenant.
* Add deployment validation to detect missing required tables before application startup or feature execution.
* Improve exception handling so missing schema objects produce explicit database errors instead of a generic transaction rollback message.

---

# Lessons Learned

## Debugging Insights

* A successful file download does not guarantee all expected artifacts were generated.
* Compare generated files on disk before investigating download logic.
* Compare behavior between a known-good patient and a failing patient to isolate data or schema differences.
* Validate tenant schema consistency when issues affect only a specific customer.

## Prevention Recommendations

* Maintain consistent schema migrations across all tenant databases.
* Include automated post-deployment schema validation checks.
* Monitor CDA generation failures with alerts for transaction rollbacks.
* Log the exact SQL exception and missing database object instead of only reporting `"Transaction silently rolled back because it has been marked as rollback-only"`.

## Operational Checklist for Similar Issues

1. Reproduce the issue through **Patient Chart → Utilities → Export Clinical Summary**.
2. Verify `CDADownloadOptionView.Action` returns encounters.
3. Confirm `GenerateCDA.Action` executes successfully.
4. Inspect `cda_Outbox` for both `.cda` and `-Clinical-Summary.xml` files.
5. Compare ZIP contents with generated files on disk.
6. Test the Spring Boot `CDAGeneration/GenerateCda` endpoint directly.
7. Review application logs for transaction rollback errors.
8. Compare the tenant schema against a known-good environment.
9. Verify the existence of `encounter_chart_app_reference_values`.
10. Apply the missing schema migration if the table is absent and revalidate end-to-end generation.
