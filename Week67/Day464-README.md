
# MIPS Measure Configuration and EMR Activity Integration

## Overview

This document captures the analysis and technical findings related to the **MIPS configuration request for the CCG account**, based on the investigation and discussion in this conversation.

The original request contained multiple independent items. This document specifically addresses **item #11**:

> **“MIPS: Need to configure the MIPS measures.”**

The investigation established that the MIPS measures are **already configured for the providers in the CCG account**. The current observation is that there are **no EMR activities recorded for the providers**, particularly no Clinical Form activity.

The expected functional flow is:

```text
MIPS Measure Configuration
        |
        v
Provider performs relevant EMR activity
        |
        v
EMR Activity is recorded
        |
        v
Activity count/frequency becomes available
        |
        v
MIPS Job processes the activity
        |
        v
MIPS measure count is updated
        |
        v
Updated count is reflected in MIPS Report
```

At the time of investigation, the configuration portion was already in place, while the required EMR activity data was not present.

---

## Background / Context

A business request was received containing **11 separate items** related to CCA functionality, including reporting, payment calculations, scheduler behavior, surveys, Connie, and MIPS.

The MIPS-specific request was:

> **“MIPS: Need to configure the MIPS measures.”**

Because the request was broad and did not specify individual measure IDs or a particular configuration defect, the first step was to verify whether the MIPS measures were actually configured for the relevant providers.

The CCG account was used for verification.

The investigation showed that MIPS measures were already configured for the providers.

Therefore, the issue should not initially be treated as a missing MIPS configuration problem.

---

## Problem Statement

### Original reported requirement

The original business request stated:

> **MIPS: Need to configure the MIPS measures.**

### Observed system state

The CCG account was reviewed and the following was observed:

* MIPS measures are already configured for the providers.
* The EMR Activity Report does not currently show relevant clinical-form activity.
* The **Clinical Forms count is 0** in the reviewed activity report.
* Therefore, there is currently no clinical-form activity/count available for the downstream MIPS processing flow.

### Important distinction

The current finding is:

```text
MIPS configuration = Present
EMR clinical activity = Not present
```

This means the absence of updated MIPS activity/counts should **not automatically be interpreted as a MIPS configuration failure**.

---

## Impact

Based on the investigation performed in the conversation:

* The providers already have MIPS measures configured.
* There are currently no relevant EMR activities recorded for the providers.
* Without relevant EMR activity, the MIPS processing flow has no corresponding activity/count to process.
* Consequently, the expected MIPS count will not change until the relevant provider activity occurs.
* Once the appropriate EMR activity is performed and recorded, the corresponding count is expected to be reflected in the MIPS report.

### Business impact

The observed state does **not indicate that MIPS configuration is missing**.

Instead, the current state indicates that there is no source EMR activity available to drive the expected MIPS count update.

---

## Environment Details

| Item                                  | Finding                                                        |
| ------------------------------------- | -------------------------------------------------------------- |
| Account investigated                  | **CCG account**                                                |
| Feature                               | MIPS                                                           |
| Provider configuration                | MIPS measures already configured                               |
| EMR activity                          | No relevant activity currently recorded                        |
| Clinical Forms activity               | `0` in the reviewed activity report                            |
| MIPS report                           | Expected to reflect activity once relevant EMR activity occurs |
| Specific performance year             | **Not specified in the conversation**                          |
| Specific MIPS measure mapping         | **Not fully specified**                                        |
| Application/module represented by CCA | **Not explicitly defined**                                     |

> **Note:** The conversation does not provide enough information to identify the exact database tables, APIs, stored procedures, service names, or job implementation details. These should be determined from the application source code and database if deeper implementation investigation is required.

---

## System Components Involved

The following logical components were identified from the discussion.

### 1. MIPS Measure Configuration

The provider has MIPS measures configured.

The screenshots/discussion indicate that multiple CMS MIPS measures were already available for the provider.

Examples observed include:

| Measure   | Description                                          |
| --------- | ---------------------------------------------------- |
| CMS156v14 | Use of High-Risk Medications in the Elderly          |
| CMS177v14 | Suicide Risk Assessment                              |
| CMS128v14 | Anti-Depressant Medication Management                |
| CMS122v14 | Diabetes: Glycemic Status Assessment Greater Than 9% |
| CMS165v14 | Controlling High Blood Pressure                      |
| CMS138v14 | Tobacco Use: Screening and Cessation Intervention    |

These examples demonstrate that the provider already has MIPS measure configuration.

> The conversation does not establish that these are the only configured measures.

---

### 2. EMR Activities

The EMR Activity Report is the next important component in the flow.

The reviewed activity report contained values such as:

| Activity                       | Count |
| ------------------------------ | ----: |
| Encounter                      |     9 |
| Clinical Forms                 |     0 |
| Frequently Used Clinical Forms |   `-` |
| Investigation                  |     7 |
| Prescription                   |     0 |
| Current Medication             |     0 |
| Assessment                     |     0 |
| Vitals                         |     0 |
| Patient Documents              |     8 |
| Refill Request                 |     0 |

The most important finding for the MIPS investigation is:

```text
Clinical Forms = 0
```

This indicates that no clinical-form activity was available in the reviewed account at the time of investigation.

---

### 3. MIPS Job

The expected flow discussed in the conversation is that the MIPS job processes relevant EMR activity.

Conceptually:

```text
EMR Activity
     |
     v
Activity Count / Frequency
     |
     v
MIPS Job
     |
     v
MIPS Measure
     |
     v
MIPS Report
```

The exact implementation of the MIPS job was **not inspected or established in the conversation**.

Therefore, the following remain implementation-level questions:

* Exact job/service name
* Job schedule
* Source table/query
* Activity-to-measure mapping
* Increment vs. recalculation behavior
* Retry/idempotency behavior
* Error handling
* Logging
* Provider filtering

These should be investigated in the codebase if implementation work is requested.

---

## Investigation Timeline

### Step 1 — Review the original request

The original email contained 11 requests.

The MIPS-specific request was item #11:

```text
11. MIPS: Need to configure the MIPS measures.
```

### Step 2 — Determine whether MIPS configuration exists

The CCG account was reviewed.

Finding:

```text
MIPS measures are already configured for providers.
```

Therefore:

```text
Configuration missing?
        |
        +-- No
```

### Step 3 — Review EMR activity

The EMR Activity Report was reviewed.

The relevant observation was:

```text
Clinical Forms = 0
```

Therefore:

```text
Relevant EMR activity available?
        |
        +-- No
```

### Step 4 — Analyze expected MIPS flow

The expected business flow was established as:

```text
Configured MIPS Measure
        |
        v
Relevant Provider EMR Activity
        |
        v
Activity Count/Frequency
        |
        v
MIPS Job
        |
        v
MIPS Measure Count
        |
        v
MIPS Report
```

### Step 5 — Determine current conclusion

The current evidence indicates that the MIPS measures are configured, but the source EMR activity has not occurred/been recorded.

Therefore, no MIPS configuration change was identified as necessary from the information available.

---

# Root Cause Analysis

## Current Root Cause

Based on the evidence discussed:

> **The MIPS measures are already configured. The reviewed CCG account currently has no relevant EMR clinical-form activity recorded, so there is no corresponding activity count available to drive the MIPS update flow.**

In simplified form:

```text
No relevant EMR activity
          |
          v
No clinical-form count
          |
          v
No activity for MIPS processing
          |
          v
MIPS count does not change
```

### Root cause classification

| Area                    | Status                           |
| ----------------------- | -------------------------------- |
| MIPS configuration      | ✅ Configured                     |
| Provider configuration  | ✅ Present                        |
| EMR activity            | ⚠️ No relevant activity observed |
| Clinical Forms activity | ⚠️ Count = 0                     |
| MIPS job failure        | ❓ Not established                |
| Database issue          | ❓ Not investigated               |
| API issue               | ❓ Not investigated               |
| UI issue                | ❓ Not established                |

> **Important:** The conversation does not provide evidence that the MIPS job itself is failing. It would be incorrect to classify the MIPS job as defective based only on the current findings.

---

## Contributing Factors

### 1. Original requirement was broad

The original request only stated:

> “Need to configure the MIPS measures.”

It did not specify:

* Measure IDs
* Provider
* Performance year
* Expected count
* Specific clinical activity
* Expected before/after values
* Specific MIPS report discrepancy

This made verification of the actual expected behavior necessary.

### 2. Configuration and activity are separate concerns

A MIPS measure can be configured without there being current clinical activity to contribute to that measure.

Therefore:

```text
Configured measure ≠ Measure has activity
```

### 3. No EMR activity at time of verification

The observed Clinical Forms count was `0`.

Consequently, the downstream MIPS flow has no corresponding activity to process based on the discussed design.

---

# Detailed Technical Findings

## MIPS Configuration

The provider already has MIPS measures configured.

Therefore, the original requirement:

> “Need to configure the MIPS measures”

appears to already be satisfied in the CCG account.

No missing configuration was identified during the discussed verification.

---

## EMR Activity

The EMR Activity Report is important because the expected MIPS update depends on provider activity.

The reviewed report showed:

```text
Clinical Forms = 0
```

This is the key technical observation.

Other EMR activity types had counts, including:

```text
Encounter = 9
Investigation = 7
Patient Documents = 8
```

However, the discussion specifically associates the MIPS update flow with relevant clinical-form activity.

Therefore, the presence of other EMR activity types should not automatically be interpreted as MIPS activity.

---

## Clinical Form Activity

The expected flow discussed was:

1. Provider performs a relevant clinical activity.
2. The EMR records the activity.
3. The activity contributes to a count/frequency.
4. The MIPS job processes the activity.
5. The corresponding MIPS measure is updated.
6. The updated count is reflected in the MIPS report.

At the time of investigation, step 2 was not producing a relevant Clinical Forms count in the reviewed account.

---

## Frequently Used Clinical Forms

The conversation mentioned:

> “If have any count and frequently used clinical form having count means we will update MIPS job.”

This indicates that **Frequently Used Clinical Forms** may have some relevance to the MIPS processing flow.

However, the exact business rule is **not fully defined**.

It is currently unclear whether:

* Any clinical-form activity should trigger MIPS processing.
* Only frequently used forms should trigger MIPS processing.
* A minimum frequency/count threshold exists.
* Specific clinical forms map to specific MIPS measures.
* Frequency is calculated over a particular period.
* Frequency is per provider, patient, encounter, or organization.

These points should be clarified before changing implementation logic.

---

# MIPS Processing Flow

The currently understood logical architecture is:

```text
+--------------------------+
| MIPS Measure             |
| Configuration            |
+------------+-------------+
             |
             | configured for provider
             v
+--------------------------+
| Provider EMR Activity    |
| / Clinical Form          |
+------------+-------------+
             |
             | activity recorded
             v
+--------------------------+
| Activity Count /         |
| Frequency                |
+------------+-------------+
             |
             | consumed by
             v
+--------------------------+
| MIPS Job                 |
+------------+-------------+
             |
             | update/recalculate
             v
+--------------------------+
| MIPS Measure Result      |
+------------+-------------+
             |
             v
+--------------------------+
| MIPS Report              |
+--------------------------+
```

---

# Expected Behavior

Suppose a MIPS measure is already configured.

Before relevant activity:

```text
MIPS Measure
Count = X
```

A provider then performs the relevant EMR activity.

The system should record:

```text
Clinical Activity
Count = N
```

The MIPS processing flow should then process that activity.

Expected result:

```text
MIPS Measure
Count = updated value
```

The updated value should then be visible in the MIPS report.

---

# Important Implementation Consideration: Increment vs. Recalculation

The conversation identified an important engineering consideration.

There are two possible implementations for updating the MIPS count.

## Model A — Incremental Update

```text
Existing MIPS Count
        +
New Activity Count
        =
Updated MIPS Count
```

Example:

```text
Existing count = 10
New activity = 3

Updated count = 13
```

### Risk

This approach can produce incorrect results if:

* The same activity is processed twice.
* A background job is retried.
* An activity is deleted or corrected.
* A provider changes an encounter/form.
* Historical activity is reprocessed.

Therefore, the implementation must be idempotent if this model is used.

---

## Model B — Recalculation

A more robust model may be:

```text
Read source EMR activities
        |
        v
Apply MIPS eligibility rules
        |
        v
Calculate numerator/denominator/count
        |
        v
Store current MIPS result
```

This reduces the risk of duplicate increments but may require more database processing.

> The conversation did **not establish which model the existing MIPS job uses**. This should be verified before making any code changes.

---

# SQL Analysis and Scripts

## Investigation Queries

**No SQL queries or scripts were provided in the conversation.**

Therefore, there are no SQL scripts to document, classify, or recommend for execution.

The exact database objects supporting the following areas remain unknown:

* MIPS configuration
* Provider/MIPS mapping
* EMR activity
* Clinical form activity
* Frequently used clinical forms
* MIPS job processing
* MIPS report results

### Recommended future database investigation

If source-level investigation becomes necessary, identify:

```text
MIPS configuration table(s)
        |
        +-- Provider mapping
        |
        +-- Measure definition
        |
        +-- Measure configuration

EMR activity table(s)
        |
        +-- Provider
        +-- Patient
        +-- Encounter
        +-- Activity type
        +-- Clinical form
        +-- Activity date
        +-- Count/frequency

MIPS result table(s)
        |
        +-- Measure
        +-- Provider
        +-- Numerator
        +-- Denominator
        +-- Count
        +-- Reporting period
```

No database names should be assumed until verified against the application schema.

---

# Code Changes

## Current Investigation

No code changes were discussed or implemented.

No files, classes, services, APIs, stored procedures, or background-job source code were identified in the conversation.

### Status

```text
Code change required: Not established
Configuration change required: Not established
MIPS job modification: Not established
Database fix: Not established
```

The current conclusion is based on configuration and UI/activity verification.

---

# Fixes and Workarounds

## Permanent Fix

No software fix was identified as necessary based on the current investigation.

The existing behavior is expected to depend on relevant provider EMR activity.

### Expected operational action

A provider should perform the relevant EMR activity.

Then verify:

```text
EMR activity recorded
        |
        v
Activity count updated
        |
        v
MIPS job processes activity
        |
        v
MIPS report reflects count
```

---

## Workaround

No workaround was identified or required.

The current recommendation is to generate/perform the appropriate clinical activity and verify the downstream processing.

---

# Validation and Testing

## Current Validation

The following validation was performed:

### MIPS configuration

Verified the CCG account and confirmed that MIPS measures are already configured for providers.

**Result:**

```text
PASS
```

### EMR Activity Report

Reviewed the EMR activity report.

Relevant observation:

```text
Clinical Forms = 0
```

**Result:**

```text
No relevant clinical-form activity observed
```

### MIPS report behavior

The expected behavior discussed is:

```text
Provider performs relevant EMR activity
        |
        v
Activity count becomes available
        |
        v
MIPS processing
        |
        v
MIPS report reflects corresponding count
```

However, an end-to-end test with a newly generated clinical-form activity was **not documented as having been executed in the conversation**.

Therefore, end-to-end MIPS job validation remains an area for future testing if required.

---

# Recommended Test Scenarios

For future validation, the following scenarios should be tested.

## Test 1 — Existing configuration

**Steps:**

1. Select provider.
2. Open MIPS configuration.
3. Verify configured measures.
4. Confirm measure is enabled/available.

**Expected:**

```text
MIPS measure is configured.
```

---

## Test 2 — No clinical activity

**Condition:**

```text
Clinical Forms = 0
```

**Expected:**

* No new clinical-form-driven MIPS activity.
* MIPS count should not unexpectedly increase.

---

## Test 3 — New relevant clinical-form activity

**Steps:**

1. Select provider.
2. Perform the relevant clinical form/activity.
3. Save/complete the activity.
4. Verify EMR Activity Report.
5. Verify Clinical Forms count.
6. Allow/run the MIPS processing job.
7. Open MIPS report.

**Expected:**

```text
Clinical Form count increases
        ↓
MIPS processing occurs
        ↓
Corresponding MIPS count is reflected
```

---

## Test 4 — Multiple activities

Perform the same relevant activity multiple times.

Verify that:

```text
Activity count = expected number
MIPS count = expected corresponding value
```

---

## Test 5 — Job retry / duplicate processing

If the MIPS job is retryable, process the same activity more than once.

Verify that the MIPS count does not become incorrectly duplicated.

This is particularly important if the implementation uses incremental updates.

---

## Test 6 — Multiple providers

Verify that activity generated by Provider A does not incorrectly affect Provider B's MIPS measure.

---

## Test 7 — Multiple clinical forms

If different clinical forms map to different measures, verify that each activity updates only the appropriate measure.

---

# Risks and Side Effects

## Data Integrity Risk

If MIPS counts are incremented directly from activity records without duplicate protection, retrying a background job could potentially result in double counting.

This is an implementation consideration and was **not observed as an existing defect** in the conversation.

---

## Mapping Risk

If clinical forms are mapped incorrectly to MIPS measures, valid activity could be attributed to the wrong measure.

The mapping logic should therefore be explicitly documented.

---

## Reporting Risk

If the MIPS report reads from a separate aggregate/result table, there may be a delay between:

```text
EMR Activity
```

and:

```text
MIPS Report
```

The expected processing interval was not specified in the conversation.

---

## Configuration Risk

MIPS measures may change by reporting/performance year.

The conversation does not specify the applicable MIPS performance year, so future maintenance should verify the correct measure version before modifying configuration.

---

# Troubleshooting Guide

## Symptom: MIPS measure is not showing an updated count

Follow this sequence.

### Step 1 — Verify MIPS configuration

Check:

```text
Provider
   ↓
MIPS Measure
   ↓
Configured?
```

If **No**, investigate configuration.

If **Yes**, continue.

---

### Step 2 — Verify EMR activity

Check:

```text
EMR Activity Report
   ↓
Clinical Forms
```

If:

```text
Clinical Forms = 0
```

then there is no relevant clinical-form activity currently available based on the reviewed report.

---

### Step 3 — Perform relevant activity

Have the provider perform the appropriate clinical activity/form.

Then check the activity report again.

Expected:

```text
Clinical Forms > 0
```

assuming the performed activity is one that should be recorded in this activity category.

---

### Step 4 — Check MIPS processing

If the activity is present but the MIPS report has not changed:

Investigate:

* MIPS job execution
* Job schedule
* Job logs
* Activity-to-measure mapping
* Provider filtering
* Reporting period
* MIPS result storage
* Processing errors

---

### Step 5 — Check report

After successful processing:

```text
EMR Activity
      ↓
MIPS Job
      ↓
MIPS Measure
      ↓
MIPS Report
```

Verify that the expected count is reflected.

---

# Diagnostic Decision Tree

```text
                    MIPS count incorrect?
                             |
                             v
                  Is MIPS measure configured?
                       /             \
                     NO               YES
                     |                 |
              Configuration      Check EMR activity
                  issue                |
                                  Clinical Forms?
                                   /        \
                                 0          > 0
                                 |            |
                          No source       Check MIPS job
                           activity            |
                                               v
                                     Job processed activity?
                                          /          \
                                        NO            YES
                                        |              |
                                   Job/config       Check measure
                                      issue          mapping/result
                                                         |
                                                         v
                                                   Check MIPS report
```

---

# Pending Work

The following items remain unclear or require additional investigation if the issue proceeds beyond the current verification.

## 1. Exact MIPS Job Implementation

Need to identify:

* Job name
* Service/class
* Schedule
* Source data
* Processing logic
* Destination data
* Error handling

---

## 2. Database Schema

Need to identify the exact tables containing:

* MIPS measures
* Provider measure configuration
* EMR activities
* Clinical forms
* Frequently used clinical forms
* MIPS results

---

## 3. Clinical Form → MIPS Mapping

It is not explicitly documented in the conversation which clinical forms contribute to which MIPS measures.

This mapping should be established before troubleshooting a specific measure.

---

## 4. Frequently Used Clinical Form Logic

The phrase “frequently used clinical form” was discussed, but the exact rule is unclear.

Need to clarify:

* What defines “frequently used”?
* Is there a threshold?
* What period is used?
* Is the count provider-specific?
* Is it patient-specific?
* Does every clinical-form activity contribute?
* Does only a frequently used form contribute?

---

## 5. MIPS Performance Year

The conversation does not specify the MIPS performance/reporting year.

This should be confirmed because measure versions can change between reporting years.

---

## 6. End-to-End Validation

A full test should be performed:

```text
Provider
  ↓
Clinical Form
  ↓
EMR Activity
  ↓
Activity Count
  ↓
MIPS Job
  ↓
MIPS Measure
  ↓
MIPS Report
```

The conversation established the configuration and current activity state but did not document completion of this full end-to-end test.

---

# Communication / Resolution

The final response prepared for the original requester specifically addressed **item #11** to avoid confusion with the other 10 requests.

The communication stated that:

* The CCG account was verified.
* MIPS measures are already configured for providers.
* There are currently no EMR activities recorded for providers.
* Once relevant EMR activities are performed, corresponding counts are expected to be reflected in the MIPS report.
* A screenshot was attached as supporting evidence.
* The requester was invited to raise concerns or request clarification.

This distinction is important because the original email contained **11 separate requests**.

---

# Lessons Learned

## 1. Verify configuration before changing configuration

A request saying:

> “Need to configure MIPS measures”

does not necessarily mean configuration is missing.

Always verify the actual provider/account configuration first.

---

## 2. Separate configuration from activity

A configured MIPS measure does not automatically have a count.

The system needs relevant source activity.

```text
Configuration
     ≠
Activity
     ≠
Processed Result
```

These should be treated as separate troubleshooting layers.

---

## 3. Trace the complete data flow

For MIPS-related issues, investigate in this order:

```text
Provider Configuration
        ↓
EMR Activity
        ↓
Activity Count/Frequency
        ↓
MIPS Job
        ↓
MIPS Result
        ↓
MIPS Report
```

This prevents prematurely modifying the MIPS job when the actual issue may be missing source activity.

---

## 4. Use UI evidence to establish the initial state

The CCG account screenshots were useful because they established:

```text
MIPS configuration exists
```

and:

```text
Clinical Forms activity = 0
```

This provides concrete evidence for the current state.

---

## 5. Avoid assuming job failure

A MIPS report not changing does not automatically mean the MIPS job is broken.

Before investigating the job, verify that the job actually has valid input data.

---

## 6. Make support responses item-specific

When an email contains multiple independent requirements, explicitly identify the item being addressed.

For example:

> **Regarding item 11 – MIPS: Need to configure the MIPS measures**

This prevents the recipient from assuming that the entire request has been completed.

---

# Recommended Future Monitoring

For long-term maintainability, the MIPS processing flow would benefit from visibility into:

* Number of EMR activities discovered by each job run
* Number of activities eligible for MIPS
* Number of activities processed
* Number skipped
* Number failed
* Number of measures updated
* Provider ID
* Measure ID
* Processing timestamp
* Job execution ID
* Duplicate/retry detection

A useful operational log would conceptually look like:

```text
MIPS Job
----------------------------------------
Execution ID: <id>
Provider: <provider>
Activities Found: <count>
Eligible Activities: <count>
Processed Activities: <count>
Measures Updated: <count>
Skipped: <count>
Failed: <count>
Execution Time: <timestamp>
Status: SUCCESS / PARTIAL / FAILED
```

This would make future MIPS troubleshooting significantly easier.

---

# Current Status

| Area                                         | Status                           |
| -------------------------------------------- | -------------------------------- |
| CCG account verified                         | ✅                                |
| Provider MIPS measures configured            | ✅                                |
| EMR activities available                     | ⚠️ No relevant activity observed |
| Clinical Forms count                         | ⚠️ `0`                           |
| MIPS job failure confirmed                   | ❌ No                             |
| MIPS configuration defect confirmed          | ❌ No                             |
| Code fix identified                          | ❌ No                             |
| SQL fix identified                           | ❌ No                             |
| End-to-end activity-to-MIPS test documented  | ⚠️ Pending                       |
| Exact activity-to-measure mapping documented | ⚠️ Pending                       |
| Exact MIPS job implementation documented     | ⚠️ Pending                       |

---

# Final Conclusion

The investigation does **not** currently indicate a missing MIPS configuration.

The CCG account already has MIPS measures configured for the providers. The key finding is that the reviewed EMR Activity Report currently contains **no Clinical Forms activity (`0`)**.

The expected processing chain is:

```text
MIPS Measures Configured
          ↓
Provider Performs Relevant EMR Activity
          ↓
Clinical Form / EMR Activity Recorded
          ↓
Activity Count / Frequency Available
          ↓
MIPS Job Processes Activity
          ↓
MIPS Measure Updated
          ↓
MIPS Report Reflects Updated Count
```

Therefore, the next meaningful troubleshooting step, if an updated MIPS count is expected, is to **perform/identify a relevant provider EMR activity and verify that it is recorded in the EMR Activity Report**. If the activity is recorded but the MIPS report still does not update, the investigation should then move downstream to the **MIPS job, activity-to-measure mapping, result storage, and reporting logic**.

No SQL, source-code change, database fix, API change, or MIPS job defect was established in the conversation and should not be assumed without further technical investigation.
