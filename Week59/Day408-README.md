
# Case #247660 – EAAM: Account-Specific CPT-II Code Exclusion Using Initial Settings

---

# Overview

This document describes the implementation, investigation, root cause analysis, configuration, code changes, SQL scripts, testing, and deployment process for **Case #247660 (EAAM)**.

The customer requested that specific CPT-II codes **must not be auto-generated from the backend** and **must not be pushed into claims processing** for the **EAAM account only**.

Instead of removing these CPT-II codes globally from the application (which would affect every customer), an **account-specific configuration** was introduced using **Initial Settings**.

---

# Background / Context

## Customer Request

The customer (EAAM) requested the following CPT-II codes to **not be automatically generated**:

| CPT-II Code | Description                                       |
| ----------- | ------------------------------------------------- |
| G8421       | BMI not documented; no reason given               |
| G8419       | BMI outside normal range; no follow-up documented |
| G8428       | Medication list not documented                    |
| 1123F-8P    | ACP not documented                                |

Original customer request:

> Remove the following CPT-II codes from backend auto-generation so they are not pushed through claims processing.

---

# Business Requirement

The requested behavior should apply **only to EAAM**.

Other accounts must continue to generate these CPT-II codes normally.

---

# Problem Statement

Originally, CPT-II generation logic was common for every tenant.

Removing these CPT-II codes directly from backend logic would have:

* affected every customer
* broken existing functionality
* introduced regressions

Therefore a configurable solution was required.

---

# Impact

Without this enhancement:

* EAAM receives unwanted CPT-II codes.
* Claims include CPT-II codes that customer explicitly requested to exclude.
* Removing codes globally would negatively impact every other account.

---

# Environment

## Account

EAAM

## Module

Automatic CPT-II Generation

## Service

```
ChargesServicesImpl.java
```

Location:

```
glaceemr_backend_new
 └── services
      └── chart
           └── charges
                └── ChargesServicesImpl.java
```

---

# System Components Involved

## Java Classes

* ChargesServicesImpl
* InitialSettingsRepository

## Database

Table:

```
initial_settings
```

---

# Existing CPT-II Generation Flow

Automatic CPT-II generation occurs inside:

```
saveServicesBasedOnHedisMeasuresConfig(...)
```

Based on measure IDs, CPT codes are generated for:

| Measure | CPT Category          |
| ------- | --------------------- |
| 236     | Blood Pressure        |
| 128     | BMI                   |
| 130     | Medication Review     |
| 112     | Mammogram             |
| 134     | Depression Screening  |
| 47      | Advance Care Planning |
| 1       | Hemoglobin            |

Generated CPT codes eventually flow through:

```
processServiceChange(...)
```

where:

* existing CPT updated
* new CPT inserted
* old CPT deleted

---

# Original Approach (Rejected)

Initial implementation attempted to completely remove:

```
G8421
G8419
G8428
1123F-8P
```

from backend logic.

### Problem

Removing these codes from common logic affects every account.

Therefore this solution was rejected.

---

# Final Solution

Introduce an account-specific Initial Setting.

If a CPT code appears in the configured list:

```
Ignore cptII cptList
```

that CPT code is skipped during automatic generation.

---

# Initial Setting Configuration

Configuration Name

```
Ignore cptII cptList
```

Purpose

Stores comma-separated CPT-II codes that should not be generated.

Example:

```
G8421,G8419,G8428,1123F-8P
```

---

# Database Changes

## Insert Configuration

```sql
INSERT INTO initial_settings (
    initial_settings_option_id,
    initial_settings_option_type,
    initial_settings_option_name,
    initial_settings_option_value,
    initial_settings_view_type,
    initial_settings_visible,
    initial_settings_option_name_desc
)
VALUES (
    (SELECT MAX(initial_settings_option_id) + 1 FROM initial_settings),
    2,
    'Ignore cptII cptList',
    '',
    'text',
    true,
    'Ignore cptII cptList'
);
```

Purpose

Adds configurable ignore list.

Type

Configuration script

---

## EAAM Configuration

```sql
UPDATE initial_settings
SET initial_settings_option_value =
'G8421,G8419,G8428,1123F-8P'
WHERE initial_settings_option_name='Ignore cptII cptList';
```

Purpose

Enable CPT exclusion only for EAAM.

Type

Configuration update

---

## Validation Query

```sql
SELECT *
FROM initial_settings
WHERE initial_settings_option_name='Ignore cptII cptList';
```

Purpose

Verify configuration exists.

---

# Code Changes

## File

```
ChargesServicesImpl.java
```

---

## Step 1

Read ignored CPT list once.

Added inside:

```
saveServicesBasedOnHedisMeasuresConfig(...)
```

```java
String ignoredCptList =
    initialSettingsRepository.findVisibleOptionValueByOptionName(
        "Ignore cptII cptList");
```

Behavior

EAAM

```
ignoredCptList =
G8421,G8419,G8428,1123F-8P
```

Other accounts

```
ignoredCptList = null
```

No exception occurs because null is handled.

---

## Step 2

Updated

```
processServiceChange(...)
```

Signature

Before

```java
processServiceChange(...)
```

After

```java
processServiceChange(
    ...,
    ignoredCptList
)
```

---

## Step 3

Skip ignored CPTs

```java
if (isIgnoredCpt(newCptCode, ignoredCptList)) {
    return;
}
```

Meaning

Ignored CPTs

* not inserted
* not updated
* not processed

---

## Step 4

Added helper

```java
private boolean isIgnoredCpt(
    String cptCode,
    String ignoredCptList
)
```

Logic

```java
if(cptCode==null)
    return false;

if(ignoredCptList==null)
    return false;

Split by comma

Compare

Ignore case

Return true
```

---

# Existing CPT Categories

The helper automatically works for every category because every generated CPT passes through `processServiceChange(...)`.

## BMI

```
G8420
G8417
G8418
G2181
G8419
```

---

## Medication

```
G8427
G8428
```

---

## Depression

```
G8431
G8511
G8510
G8433
G8432
```

---

## Hemoglobin

```
3044F
3051F
3052F
3046F
```

---

## Mammogram

```
G9900
G9899
```

---

## Blood Pressure

```
3074F
3075F
3077F
3078F
3079F
3080F
```

---

## Advance Care Planning

```
1123F
1124F
```

---

# Behavior After Change

Example

Configuration

```
Ignore cptII cptList

↓

G8421,G8419,G8428,1123F-8P
```

Generated CPT

```
G8428
```

↓

```
isIgnoredCpt()

↓

true
```

↓

```
return;
```

↓

No insert

No update

No auto-generation

---

Another generated CPT

```
G8427
```

↓

```
Not present
```

↓

Generated normally.

---

# Investigation Timeline

## Initial Investigation

Customer requested removal of four CPT-II codes.

---

Initial implementation

Removed CPT generation from backend.

---

Issue identified

Global impact.

---

Discussion

Introduced account-specific configuration.

---

Implementation

Created Initial Setting.

---

Updated generation logic.

---

Testing

Validated in testing environment.

---

Commit created.

---

Beta deployment requested.

---

# Root Cause Analysis

Root Cause

The application contained a shared CPT-II generation workflow.

Customer-specific exclusions were not configurable.

Contributing Factors

* Common backend logic
* No tenant-specific exclusion mechanism
* CPT generation hardcoded

---

# Validation & Testing

Verified:

✅ Existing accounts continue to generate CPT-II normally.

✅ EAAM ignores configured CPT-II codes.

✅ Non-configured CPT-II codes still generate.

✅ No NullPointerException when configuration is absent.

### Null Handling

For accounts without the configuration:

```java
ignoredCptList == null
```

The helper safely returns:

```java
false
```

No exception occurs.

---

# Deployment Steps

1. Deploy backend.
2. Add configuration in target account.
3. Update ignored CPT list.
4. Restart application if required.
5. Test automatic CPT generation.
6. Validate claims.

---

# CRM Fixes

## Fix 1

Add configuration.

```sql
INSERT INTO initial_settings (
    initial_settings_option_id,
    initial_settings_option_type,
    initial_settings_option_name,
    initial_settings_option_value,
    initial_settings_view_type,
    initial_settings_visible,
    initial_settings_option_name_desc
)
VALUES (
    (SELECT MAX(initial_settings_option_id) + 1 FROM initial_settings),
    2,
    'Ignore cptII cptList',
    '',
    'text',
    true,
    'Ignore cptII cptList'
);
```

---

## Fix 2

Configure ignored CPT-II codes for EAAM.

```sql
UPDATE initial_settings
SET initial_settings_option_value =
'G8421,G8419,G8428,1123F-8P'
WHERE initial_settings_option_name =
'Ignore cptII cptList';
```

---

# Risks

Very Low

Only accounts containing:

```
Ignore cptII cptList
```

are affected.

Accounts without this configuration continue using the existing behavior.

---

# Future Enhancements

* Add UI validation for CPT code format.
* Support wildcard or grouped CPT exclusions.
* Provide an admin page to manage ignored CPT-II codes.
* Log skipped CPT-II codes for auditing.

---

# Commit

```
Support account-specific CPTII code exclusion using initial settings
```

**Commit ID:** `58265`

---

# Key Takeaways

* Avoid modifying shared business logic for customer-specific requests.
* Prefer configuration-driven behavior over hardcoded changes.
* A centralized helper (`isIgnoredCpt`) ensures consistent exclusion across all CPT-II categories.
* Using the `initial_settings` table enables tenant-specific customization without impacting other accounts.
* Null-safe handling of the configuration ensures accounts without the setting continue functioning without errors.
