
# CPT II Auto-Population Enhancement – Dr. Farooqui (HbA1c & Blood Pressure Measures)

## Overview

This document captures the complete investigation, implementation, validation, and communication flow related to the CPT II automation enhancement for Dr. Farooqui’s practice.

The enhancement focused on enabling and validating automatic CPT II code generation for:

* **HbA1c Measures**
* **Blood Pressure Measures**

The work was requested to improve:

* MIPS quality reporting
* Claim-based reporting
* eCQM quality capture
* Provider incentive tracking
* Managed care quality metrics

This README serves as:

* Technical implementation documentation
* Root Cause Analysis (RCA)
* Troubleshooting guide
* Support reference
* Developer onboarding reference
* Future maintenance document

---

# Background / Context

## Business Requirement

Powers Health requested automation support for CPT II quality reporting codes for Dr. Farooqui’s office.

Requested measures included:

### Blood Pressure CPT II Codes

* 3074F
* 3075F
* 3077F
* 3078F
* 3079F
* 3080F

### HbA1c CPT II Codes

* 3044F
* 3045F
* 3046F
* 3051F
* 3052F

The goal was to ensure:

* CPT II codes are automatically generated
* Providers receive proper MIPS quality credit
* Claims include CPT II reporting measures
* Manual billing entry is minimized

---

# Problem Statement

## Original Issue

Dr. Farooqui’s office reported that:

* HbA1c and BP quality measures were not generating CPT II services automatically.
* Providers were not receiving proper MIPS quality credit.
* Some lab results existed only on paper/email and were not reflected in EHR structured data.
* CPT II claim reporting was inconsistent.

---

# Impact

## Business Impact

### Without CPT II automation:

* Providers lose MIPS quality credit
* Managed care incentives are affected
* Claims lack required quality measure codes
* Manual workflows increase operational burden

### Technical Impact

* CPT II services were missing from:

  * Superbill
  * Claim reporting
  * Service detail records

---

# Environment Details

| Component      | Details                      |
| -------------- | ---------------------------- |
| Product        | Glace EHR                    |
| Module         | CPT II Auto Population       |
| Provider       | Dr. Farooqui                 |
| Feature Area   | MIPS / eCQM / Claims         |
| Database       | PostgreSQL (`mpcare`)        |
| Reporting Type | Claim-based CPT II Reporting |

---

# System Components Involved

## Database Tables

| Table                  | Purpose                                          |
| ---------------------- | ------------------------------------------------ |
| `service_detail`       | Stores encounter services/CPT entries            |
| `cpt`                  | CPT master table                                 |
| `leaf_patient`         | Encounter/template information                   |
| `patient_registration` | Patient registration/account details             |
| `labdata`              | Lab results storage (assumed from investigation) |

---

# Investigation Timeline

## Phase 1 – Initial Requirement Gathering

### Customer Request

Powers Health requested:

* Automatic CPT II generation
* Claim inclusion for quality reporting
* Support for:

  * HbA1c
  * Blood Pressure measures

---

## Phase 2 – Existing Workflow Analysis

Initial observations:

### HbA1c Logic

* HbA1c could already be captured through:

  * eCQM review workflow
  * flowsheet/manual entry

### Problem

The workflow was dependent on:

* structured EHR documentation
* encounter date alignment
* lab review flow

---

## Phase 3 – Feasibility and Logic Review

### Important Discovery

CPT II auto-population required:

| Condition           | Required   |
| ------------------- | ---------- |
| Encounter Date      | Must match |
| Performed Date      | Must match |
| Result Availability | Required   |
| Template Sign-off   | Required   |

Mismatch between encounter date and lab performed date prevented CPT II generation.

---

## Phase 4 – Enhancement Implementation

### Implemented Changes

The old:

* M-code
* G-code logic

was updated to:

* CPT II code logic

for:

* HbA1c
* Blood Pressure measures

---

# Root Cause Analysis

## Primary Root Cause

CPT II services were not generated because the required automation conditions were not met.

---

## Detailed Root Causes

### 1. Missing HbA1c Results

For several encounters:

* HbA1c lab results were unavailable in the system.

Without result availability:

* CPT II logic could not determine measure qualification.

---

### 2. Unsigned Templates

Blood Pressure CPT II generation depended on:

* template sign-off OR
* Auto Populate execution

Several encounters were:

* unsigned
* not auto-populated

Therefore:

* CPT II services were never inserted into `service_detail`.

---

### 3. Workflow Dependency

Automation relied on:

* structured EHR data
* correct save/sign workflow
* matching dates

Manual or paper workflows bypassed automation triggers.

---

# Detailed Technical Findings

## CPT II Auto Population Logic

### Blood Pressure Logic

CPT II services are generated when:

* Template is signed
  OR
* User clicks “Auto Populate”

### HbA1c Logic

#### Manual Entry Flow

If result entered manually:

* CPT II service generated during save call

#### Result Parsing Flow

If result received through parser:

* CPT II generated automatically using result date

---

# SQL Analysis and Scripts

## Investigation Queries

---

### Query 1 – Patient Service History

```sql
select
    sd.service_detail_patientid,
    sd.service_detail_dos,
    c.cpt_cptcode
from service_detail sd
join cpt c
    on c.cpt_id = sd.service_detail_cptid
where sd.service_detail_patientid = 16658
order by sd.service_detail_dos desc;
```

## Purpose

Retrieve all CPT/service history for patient.

## Tables Involved

* `service_detail`
* `cpt`

## Findings

No CPT II codes existed for expected 2026 encounters.

---

## Query 2 – CPT II Validation Query

```sql
select
    sd.service_detail_patientid,
    sd.service_detail_dos,
    c.cpt_cptcode
from service_detail sd
join cpt c
    on c.cpt_id = sd.service_detail_cptid
where sd.service_detail_patientid = 16658
and c.cpt_cptcode in
(
    '3044F','3045F','3046F','3051F','3052F',
    '3074F','3075F','3077F',
    '3078F','3079F','3080F'
)
order by sd.service_detail_dos desc;
```

## Purpose

Validate whether CPT II services were inserted.

## Result

No qualifying CPT II records found.

---

## Query 3 – Encounter vs Service Validation

```sql
select
    lp.leaf_patient_patient_id,
    lp.leaf_patient_created_date::date as encounter_date,
    sd.service_detail_dos,
    c.cpt_cptcode
from leaf_patient lp
left join service_detail sd
    on sd.service_detail_patientid = lp.leaf_patient_patient_id
left join cpt c
    on c.cpt_id = sd.service_detail_cptid
where lp.leaf_patient_patient_id = 16658
and c.cpt_cptcode in
(
    '3044F','3045F','3046F','3051F','3052F',
    '3074F','3075F','3077F',
    '3078F','3079F','3080F'
)
order by encounter_date desc;
```

## Purpose

Compare encounters against generated CPT II services.

## Observation

Encounter existed, but CPT II services absent.

---

## Query 4 – HbA1c Result Validation

```sql
select *
from labdata
where labdata_patientid = 16658
and lower(labdata_testname) like '%a1c%';
```

## Purpose

Verify whether HbA1c result existed.

## Finding

Used to confirm result availability dependency.

---

## Query 5 – Encounter Status Validation

```sql
select
    leaf_patient_patient_id,
    leaf_patient_created_date,
    leaf_patient_status
from leaf_patient
where leaf_patient_patient_id = 16658
order by leaf_patient_created_date desc;
```

## Purpose

Validate signed encounter/template status.

---

# SQL Error Observed

## Error

```sql
ERROR:  missing FROM-clause entry for table "pr"
```

## Root Cause

Query referenced:

```sql
pr.patient_registration_accountno
```

But:

* `patient_registration pr`
  table was never joined.

## Resolution

Either:

* remove the reference
  OR
* add proper join.

---

# Code Changes

## Implemented Changes

### CPT II Mapping Updates

Old:

* M-series codes
* G-series codes

Updated to:

* CPT II code mappings

---

## Auto Population Logic

### Before

* Limited automation
* Manual dependency
* Inconsistent generation

### After

* CPT II generated during:

  * Auto Populate
  * Template sign-off
  * Save call (HbA1c manual entry)
  * Result parsing

---

# Fixes and Workarounds

## Permanent Fix

### Implemented

* CPT II mapping enhancement
* Auto-population support

---

## Temporary Workaround

Users can manually:

* click “Auto Populate”
  before sign-off.

---

# Validation and Testing

## Validation Steps

### Database Validation

Checked:

* `service_detail`
* CPT II code insertion
* encounter dates
* signed status

---

## Functional Validation

### Tested Flows

| Scenario               | Result           |
| ---------------------- | ---------------- |
| Auto Populate clicked  | CPT II generated |
| Template signed        | CPT II generated |
| HbA1c manually entered | CPT II generated |
| Result parsing flow    | CPT II generated |
| Missing result         | No CPT II        |
| Unsigned template      | No CPT II        |

---

# Risks and Side Effects

## Risks

### 1. Data Dependency

Automation depends on:

* structured data availability

### 2. Workflow Dependency

Unsigned encounters prevent service creation.

### 3. Historical Encounters

Past encounters may not auto-backfill.

---

# Pending Work

## Open Items

### Past Encounter Verification

Further review required for:

* historical encounter handling
* possible backfill expectations

### Confirmation Pending

Need revalidation for:

* 2026 encounters
* signed template scenarios

---

# Lessons Learned

## Technical Insights

### 1. Automation Requires Workflow Completion

Template sign-off is critical.

### 2. Structured Data Matters

Paper/email workflows bypass automation.

### 3. Date Alignment Matters

Encounter date and result date synchronization is essential.

---

# Troubleshooting Guide

## CPT II Not Generated?

### Check 1 – Is Template Signed?

If not:

* CPT II generation will not occur.

---

### Check 2 – Was Auto Populate Used?

If not:

* manually trigger it.

---

### Check 3 – Does HbA1c Result Exist?

Verify lab result availability.

---

### Check 4 – Do Encounter and Result Dates Match?

Mismatch can prevent generation.

---

# Recommended Monitoring

## Suggested Improvements

### Logging

Add logs for:

* CPT II generation trigger
* save call execution
* sign-off execution
* result parsing

### Alerting

Track:

* failed CPT II insertion attempts

### Reporting

Create audit report for:

* encounters missing CPT II measures

---

# Final Outcome

## Status

| Item                        | Status               |
| --------------------------- | -------------------- |
| CPT II code changes         | Completed            |
| Validation                  | Completed            |
| Auto-population             | Working              |
| Historical encounter review | Pending verification |

---

# Final Technical Conclusion

The CPT II enhancement was successfully implemented and validated.

The missing CPT II services for historical encounters were not caused by code failure. They were due to unmet workflow/data conditions:

* Missing HbA1c results
* Unsigned BP templates
* Auto Populate not triggered

The system is functioning correctly for future encounters when the required workflow steps are followed.
