
# MIPS 2026 – Investigation of Mandatory Sign Validation for Smoking & Blood Pressure Documentation

---

# Overview

This document captures the complete investigation performed for the customer request related to **MIPS 2026** in the **Nassir** environment.

The investigation focused on determining whether the existing **MIPS Sign Alert** functionality could satisfy a new customer requirement that prevents providers from signing progress notes when mandatory MIPS documentation (Smoking Status and Blood Pressure) is missing.

The investigation included:

- Requirement analysis
- Existing feature analysis
- Configuration verification
- Functional testing
- Gap analysis
- Business requirement validation
- Enhancement feasibility assessment

This document serves as:

- Technical Knowledge Base
- Root Cause Analysis (RCA)
- Troubleshooting Guide
- Functional Documentation
- Future Enhancement Reference

---

# Background / Context

## Customer Request

**Subject**

> MIPS 2026: Dr Hilton & Dr Nassir

### Business Requirement

The customer requested:

> **Do not let providers sign progress notes if Smoking details and Blood Pressure (BP) values are not documented.**

Unlike an informational warning, the customer expects the application to **prevent the sign operation** until mandatory documentation is completed.

---

# Existing Functionality

Before implementing any enhancement, an investigation was performed to determine whether the product already supports this behavior.

The product already contains an existing feature named:

> **MIPS Sign Alert**

This feature is configurable at two different levels.

---

## Practice Level Configuration

Navigation:

```text
Configure
    ↓
Practice Setup
    ↓
CNM Settings
    ↓
MIPS Sign Alert
```

Available options:

- MIPS sign alert while signing template
- MIPS sign alert in chart page

---

## Provider Level Configuration

Navigation:

```text
Configure
    ↓
Meaningful Use
    ↓
QPP Configuration
```

Available option:

```text
Click this to enable MIPS alert while signing template
```

---

# Problem Statement

Initially it was unknown whether the existing MIPS Sign Alert:

- only displays warnings

or

- completely blocks signing.

The investigation aimed to answer:

> Can the existing configuration satisfy the customer's requirement without code changes?

---

# Business Impact

Customer expectation:

- Mandatory documentation before signing.

Current implementation (unknown at investigation start):

- Possibly only warning based.

Impact if unsupported:

- Providers may continue signing incomplete MIPS documentation.
- Quality measures become incomplete.
- MIPS compliance may be affected.
- Product enhancement required.

---

# Environment Details

| Item | Value |
|----------|---------|
| Account | Nassir |
| Environment | Test |
| Provider | Test Doctor |
| Reporting Years Tested | 2017 & 2026 |
| Module | CNM Settings |
| Module | QPP Configuration |
| Feature | MIPS Sign Alert |
| Screen | Progress Notes |
| Screen | Patient MIPS Status |

---

# System Components Involved

## UI Modules

- Configure
- Practice Setup
- CNM Settings
- Meaningful Use
- QPP Configuration
- Progress Notes
- Patient MIPS Status Popup

---

## Configuration

### Practice Configuration

```
CNM Settings

MIPS sign alert while signing template
```

Purpose:

Enables MIPS popup at practice level.

---

### Provider Configuration

```
QPP Configuration

Click this to enable MIPS alert while signing template
```

Purpose:

Enables MIPS popup for specific provider.

---

# Investigation Timeline

---

## Step 1 – Requirement Analysis

Received customer request.

```
Do not let providers sign notes
if

Smoking details missing

OR

Blood Pressure missing
```

Initial assumption:

Maybe existing MIPS Sign Alert already supports this.

---

## Step 2 – Documentation Review

Reviewed internal documentation for MIPS Sign Alert.

Observed workflow:

```
Click Sign

↓

Patient MIPS Status Popup

↓

Continue Signing

↓

Encounter Signed
```

Observation:

Popup appears to be informational only.

No documentation indicated that signing is blocked.

---

## Step 3 – Configuration Review

Verified provider configuration.

Observed:

### Provider Configuration

```
QPP Configuration

✔ Enable MIPS alert while signing template
```

Provider configuration was enabled.

---

Verified Practice configuration.

Observed:

```
CNM Settings

☐ MIPS sign alert while signing template
```

Practice level configuration was disabled.

---

## Initial Finding

Even though provider configuration was enabled,

Practice configuration remained disabled.

Possible reason why popup was not appearing.

---

## Step 4 – Initial Functional Test

Test Scenario:

- BP intentionally left blank.
- Clicked Sign.

Expected:

Popup.

Actual:

- No popup.
- Encounter signed successfully.

Observation:

Alert did not appear.

Initially suspected:

- BP validation issue
- Configuration issue
- Alert not working

---

## Step 5 – MIPS Status Investigation

Opened:

Patient MIPS Status.

Observed:

```
Reporting Year : 2017
```

Every quality measure displayed:

```
N/A
```

Including:

- Tobacco Use
- High Blood Pressure
- Depression
- Medication Documentation
- Screening for High Blood Pressure

---

### Finding

Patient did not qualify for any measure.

No Not Met measures existed.

Therefore:

Nothing existed for popup to display.

---

## Step 6 – Reporting Year Validation

Verified:

Provider configuration

Reporting Year

Assigned measures

Discovered mismatch during testing.

Some tests used:

- Reporting Year 2026

while encounter belonged to

- 2017

This explained why every measure showed:

```
N/A
```

---

## Step 7 – Practice Configuration Enabled

Enabled:

```
CNM Settings

✔ MIPS sign alert while signing template
```

Saved configuration.

---

## Step 8 – Functional Testing After Enabling

Clicked:

```
Sign
```

Result:

Patient MIPS Status popup displayed successfully.

Displayed measures:

- Documentation of Current Medications
- Depression Screening
- Tobacco Use Screening
- Tobacco Use (Criteria-3)
- Screening for High Blood Pressure and Follow-Up Documented

All displayed as:

```
NOT MET
```

---

## Step 9 – Popup Behavior Validation

Observed popup buttons.

Available actions:

```
Continue Signing

Close
```

Clicked:

```
Continue Signing
```

Encounter signed successfully.

---

## Final Investigation Finding

The popup functions correctly.

However,

it only warns the provider.

It does **not prevent signing.**

---

# Root Cause Analysis

## Customer Expectation

Customer expects:

```
Mandatory validation
```

---

## Existing Implementation

Current implementation performs:

```
Informational Warning
```

Only.

---

## Why popup initially did not appear

Two contributing factors.

### 1. Practice configuration disabled

```
CNM Settings

MIPS sign alert while signing template

Disabled
```

Popup skipped.

---

### 2. Testing performed against non-applicable encounter

Reporting Year:

2017

Measures:

```
N/A
```

Therefore:

Popup had no actionable measures.

---

## Actual Root Cause

The system is functioning as designed.

The gap is functional.

Current feature supports:

```
Alert
```

Customer requires:

```
Mandatory validation
```

---

# Detailed Technical Findings

## Existing Workflow

```
Provider clicks Sign

↓

Run MIPS Evaluation

↓

Display Patient MIPS Status

↓

Provider reviews Not Met measures

↓

Continue Signing

↓

Encounter Signed
```

---

## Required Workflow

Customer expects:

```
Provider clicks Sign

↓

Validate Smoking Status

↓

Validate Blood Pressure

↓

Missing?

↓

YES

↓

Prevent Signing

↓

Display Validation

↓

Return to Progress Note
```

---

## Existing Popup

The popup lists quality measures and displays statuses.

Observed Not Met measures:

- Documentation of Current Medications
- Depression Screening
- Tobacco Use Screening
- Tobacco Use Screening (Criteria-3)
- Screening for High Blood Pressure and Follow-Up Documented

Current popup still allows:

```
Continue Signing
```

---

## Configuration Dependencies

### Practice Level

```
CNM Settings

MIPS sign alert while signing template
```

Required to display popup.

---

### Provider Level

```
QPP Configuration

Enable MIPS alert while signing template
```

Required for provider.

---

Both configurations must be enabled.

---

# SQL Analysis and Scripts

## Investigation Queries

```sql
-- No SQL queries were discussed during this investigation.
```

---

## Validation Queries

None.

---

## Migration Scripts

None.

---

## Cleanup Scripts

None.

---

# Code Analysis

## Code Changes

No source code modifications were made during this investigation.

Only configuration changes and functional validation were performed.

---

## Potential Enhancement

Current flow:

```
Click Sign

↓

Popup

↓

Continue Signing
```

Expected enhancement:

```
Click Sign

↓

Validate Smoking

↓

Validate BP

↓

Missing?

↓

Block Sign

↓

Display Validation Message
```

---

# Configuration Changes

## Before

Practice configuration:

```
MIPS sign alert while signing template

Disabled
```

Provider configuration:

```
Enabled
```

---

## After

Practice configuration:

```
Enabled
```

Provider configuration:

```
Already Enabled
```

---

# Validation & Testing

## Test Case 1

### Scenario

Practice configuration disabled.

Expected:

No popup.

Actual:

No popup.

Result:

PASS

---

## Test Case 2

### Scenario

Practice configuration enabled.

Expected:

Popup displayed.

Actual:

Popup displayed successfully.

Result:

PASS

---

## Test Case 3

### Scenario

Patient with incomplete Smoking/BP documentation.

Expected:

Popup displays Not Met measures.

Actual:

Popup displayed.

Measures marked Not Met.

Result:

PASS

---

## Test Case 4

### Scenario

Continue Signing selected.

Expected:

Encounter signs successfully.

Actual:

Encounter signed successfully.

Result:

PASS

---

# Workaround

Current workaround:

Enable:

```
CNM Settings

↓

MIPS sign alert while signing template
```

This informs providers about missing documentation.

It does **not** enforce completion.

---

# Permanent Fix Recommendation

Current feature cannot satisfy customer requirement.

Recommended enhancement:

Before signing:

1. Validate Smoking Status.

2. Validate Blood Pressure.

3. If either missing:

```
Stop Sign Operation
```

Display:

```
Smoking documentation is mandatory.

Blood Pressure documentation is mandatory.

Please complete documentation before signing.
```

Only allow signing after validation passes.

---

# Risks & Side Effects

If enhancement is implemented:

Potential impacts:

- Existing provider workflow changes.
- Practices relying on warning-only behavior may be affected.
- Additional validation may require configuration to enable/disable.
- Backward compatibility should be considered.

---

# Pending Work

- Confirm whether enhancement applies:
  - Only Dr Hilton & Dr Nassir
  - Entire practice
  - All customers
- Identify Sign controller/service.
- Implement blocking validation.
- Decide whether Continue Signing button should be removed or disabled when mandatory measures fail.

---

# Lessons Learned

## Configuration Verification

Always verify both:

- Practice configuration
- Provider configuration

before debugging code.

---

## Measure Applicability

Testing with encounters outside applicable reporting criteria may result in all measures being marked **N/A**, preventing meaningful validation.

---

## Warning vs Enforcement

An informational popup does not satisfy a business requirement that explicitly requires blocking user actions. Functional requirements should distinguish between advisory alerts and mandatory validations.

---

# Conclusion

The investigation confirmed that the existing **MIPS Sign Alert** feature functions correctly when enabled at both the practice and provider levels. It displays a **Patient MIPS Status** popup highlighting **Not Met** measures, including Smoking and Blood Pressure-related quality measures.

However, the current implementation is **advisory only**. Providers can still click **Continue Signing** and complete the encounter.

Therefore, the customer's request:

> **"Do not let the providers sign the notes if the Smoking details and BP values are not documented in the progress note."**

**cannot be fulfilled through existing configuration alone** and requires a **product enhancement** that introduces mandatory validation before the sign operation is completed.
