
# README – EAAM Account: Account-Specific CPT-II Code Ignore Configuration

## Overview

This document describes the implementation performed for the **EAAM account** to support **account-specific exclusion of selected CPT-II codes** from automatic backend CPT generation.

Instead of removing CPT-II code generation globally, a configurable **Initial Setting** was introduced. This allows individual accounts to specify CPT-II codes that should be ignored during automatic generation while keeping the existing behavior unchanged for all other accounts.

---

# Background

## Customer Request

Customer: **EAAM**

The customer requested that the following CPT-II codes should **not be auto-generated** from the backend and therefore should **not be pushed to claims processing**.

| CPT Code | Description                                       |
| -------- | ------------------------------------------------- |
| G8421    | BMI not documented; no reason given               |
| G8419    | BMI outside normal range; no follow-up documented |
| G8428    | Medication list not documented                    |
| 1123F-8P | ACP not documented; reason not specified          |

Original request:

> Remove the above CPT codes from backend auto-generation.

---

# Problem

Initially, the implementation removed these CPT codes directly from the backend.

Example:

* BMI logic
* Medication Review logic
* ACP logic

However, this approach affects **every account** because the CPT generation logic is common.

Only **EAAM** requested this customization.

Therefore a global code removal is not acceptable.

---

# Solution

Instead of changing the business logic globally,

Implemented an **account-specific Initial Setting**.

If an account configures CPT-II codes inside the setting,

those codes will be ignored during automatic CPT generation.

Other accounts continue with existing behavior.

---

# Design

## New Initial Setting

Configuration Name

```
Ignore cptII cptList
```

Purpose

Stores comma-separated CPT-II codes which should be skipped during automatic CPT generation.

Example

```
G8421,G8419,G8428,1123F-8P
```

---

# Database Changes

## Step 1

Add new Initial Setting.

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

## Step 2

For EAAM account only

Configure ignored CPT-II codes.

```sql
UPDATE initial_settings
SET initial_settings_option_value =
'G8421,G8419,G8428,1123F-8P'
WHERE initial_settings_option_name='Ignore cptII cptList';
```

---

# Java Changes

File

```
glaceemr_backend_new

ChargesServicesImpl.java
```

---

## Step 1

Read configured CPT list from Initial Settings.

```java
String ignoredCptList =
initialSettingsRepository.findVisibleOptionValueByOptionName(
    "Ignore cptII_cptList"
);
```

If configuration is absent

```
ignoredCptList = null
```

No exception occurs.

---

## Step 2

Create helper method

```java
private boolean isIgnoredCpt(String cptCode,
                             String ignoredCptList)
```

Purpose

Checks whether generated CPT exists inside configured ignore list.

Implementation

```java
private boolean isIgnoredCpt(String cptCode,
                             String ignoredCptList){

    if(cptCode==null || cptCode.isEmpty())
        return false;

    if(ignoredCptList==null ||
       ignoredCptList.trim().isEmpty())
        return false;

    return Arrays.stream(
            ignoredCptList.split(","))
            .map(String::trim)
            .anyMatch(code ->
                code.equalsIgnoreCase(cptCode));
}
```

---

## Step 3

Update processServiceChange()

Old flow

```
Generate CPT

↓

Insert/Update/Delete
```

New flow

```
Generate CPT

↓

Check Ignore List

↓

Ignored ?

YES
    return

NO
    Continue existing flow
```

Implementation

```java
if (isIgnoredCpt(newCptCode, ignoredCptList)) {
    return;
}
```

This was added before

* insert
* update

logic.

---

## Step 4

Special handling for G8756

G8756 is added directly.

Original code

```java
newCptCodes.add(noBpCode);
```

Modified to

```java
if (!isIgnoredCpt(noBpCode,
                  ignoredCptList)) {
    newCptCodes.add(noBpCode);
}
```

---

# Existing Flow

```
Measure

↓

Generate CPT

↓

processServiceChange()

↓

Insert
Update
Delete
```

---

# New Flow

```
Measure

↓

Generate CPT

↓

processServiceChange()

↓

isIgnoredCpt()

↓

YES

Return

↓

No CPT generated


NO

↓

Existing Insert
Update
Delete logic
```

---

# Impact on Existing Measures

## BMI

Possible generated codes

```
G8420

G8417

G8418

G8419
```

If

```
G8419
```

is configured

↓

Skipped

Other BMI codes continue normally.

---

## Medication Review

Possible

```
G8427

G8428
```

If

```
G8428
```

configured

↓

Skipped

G8427 continues.

---

## ACP

Possible

```
1123F

1124F
```

If configured CPT matches

↓

Skipped

---

## Depression

No impact.

---

## Mammogram

No impact.

---

## Hemoglobin

No impact.

---

## Blood Pressure

No impact unless configured.

---

# Why This Design?

Without configuration

```
All Accounts

↓

Normal behavior
```

EAAM

```
Ignore List configured

↓

Only configured CPTs skipped
```

Future account

```
Different ignore list

↓

Different behavior
```

No Java changes required.

---

# Benefits

* No global code modification.
* Account-specific customization.
* Easy to maintain.
* No impact on existing accounts.
* Future accounts can reuse the same configuration.
* No deployment required for future CPT ignore requests; only database configuration is needed.

---

# Testing Performed

## Scenario 1

Configuration empty

Expected

```
All CPT-II codes generated.
```

Result

✅ Passed

---

## Scenario 2

```
Ignore List

G8421,G8419,G8428,1123F-8P
```

Expected

```
Configured CPTs skipped.
```

Result

✅ Passed

---

## Scenario 3

BMI generates

```
G8419
```

Expected

```
Skipped.
```

Result

✅ Passed

---

## Scenario 4

Medication Review generates

```
G8428
```

Expected

```
Skipped.
```

Result

✅ Passed

---

## Scenario 5

BMI generates

```
G8420
```

Expected

```
Inserted normally.
```

Result

✅ Passed

---

## Scenario 6

Hemoglobin

Expected

```
No change.
```

Result

✅ Passed

---

# CRM Fixes

## Fix 1

Run in **all AII accounts**

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

Run only in **EAAM**

```sql
UPDATE initial_settings
SET initial_settings_option_value =
'G8421,G8419,G8428,1123F-8P'
WHERE initial_settings_option_name =
'Ignore cptII cptList';
```

---

# Commit Details

**Commit ID**

```
58265
```

**Commit Summary**

> Support account-specific CPT-II code exclusion using Initial Settings.

---

# Key Takeaways

* The customer requested exclusion of selected CPT-II codes only for the EAAM account.
* Removing CPT generation globally would affect all customers.
* A configurable Initial Setting (`Ignore cptII cptList`) was introduced to provide account-specific behavior.
* The solution preserves existing functionality for all other accounts while allowing individual accounts to control which CPT-II codes are ignored through configuration alone.
