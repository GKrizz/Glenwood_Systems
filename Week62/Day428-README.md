# CCDA Treatment Plan Enhancement – Depression Follow-up Entry Implementation

---

# Overview

This document captures the implementation, investigation, debugging, and validation performed to include **Depression Follow-up** interventions in the **CCDA Treatment Plan Section**.

The enhancement adds depression follow-up clinical elements from **Patient Clinical Elements** into the CCDA **Treatment Plan (LOINC 18776-5)** using the **Instruction Activity Template**.

This document serves as:

* Technical implementation guide
* Root Cause Analysis (RCA)
* Developer reference
* Support documentation
* Future maintenance guide

---

# Background / Context

## Business Requirement

The customer required that **Depression Follow-up interventions** entered in the EMR should also appear in the generated CCDA.

Previously:

* Treatment Plan contained Care Plan Instructions.
* Depression Follow-up interventions were stored in Patient Clinical Elements.
* These follow-up interventions were **not exported** into CCDA.

Requirement:

Export Depression Follow-up interventions into:

* Human readable CCDA
* Structured CCDA XML

under

```
Treatment Plan
```

using

```
Instruction Activity
```

---

# Problem Statement

Depression Follow-up data existed in the EMR database but was missing from the generated CCDA.

Symptoms:

* Depression Follow-up visible in EMR.
* Not present in CCDA XML.
* Not rendered in HTML output.

---

# Impact

Without this enhancement:

* Depression Follow-up interventions were omitted.
* Clinical summary became incomplete.
* External systems consuming CCDA would not receive Follow-up information.

---

# Environment Details

Legacy Application

```
glacelegacy_master
```

Important package

```
com.glenwood.glaceemr.cda
```

Generation Action

```
GenerateXDAAction
```

Main Handler

```
PlanofCareSectionHandlerUtil
```

---

# System Components Involved

## Java Classes

### Modified

```
Instruction.java
```

```
PlanofCareSectionHandlerUtil.java
```

---

### Existing DTO

```
PlanOfCareDTO
```

New property used

```
DepressionFollowupList
```

---

### CDA Generation

```
GenerateXDAAction
```

↓

```
CodeGeneratorUtility.CreateCCD()
```

↓

```
PlanofCareSectionHandlerUtil
```

↓

Treatment Plan XML

---

# Database Objects

Tables involved

```
patient_clinical_elements
```

```
clinical_elements
```

```
encounter
```

---

# Investigation Timeline

## Step 1

Requirement received to include Depression Follow-up.

---

## Step 2

Investigated existing Treatment Plan generation.

Found existing support for

* Care Plan
* Instructions
* Procedures
* Referrals
* Labs
* Appointments

No support existed for Depression Follow-up.

---

## Step 3

Created DAO query.

Console output

```
===== getDepressionFollowup() called =====
```

---

## Step 4

Executed SQL.

Records successfully returned.

Console

```
Depression Follow-up Found

Instruction :
Follow-up for depression - adult

SNOMED
88848003
```

---

## Step 5

Mapped ResultSet into

```
Instruction.java
```

Added

```
ClinicalElementName
```

```
Snomed
```

```
Gwid
```

---

## Step 6

Stored list into

```
PlanOfCareDTO
```

Console

```
After DAO

DepressionFollowupList size = 2
```

---

## Step 7

Updated Narrative Block.

Generated

```
Depression Follow-up :
Follow-up for depression - adult
```

under

Treatment Plan.

---

## Step 8

Implemented Structured XML generation.

Created

```
formDepressionFollowupAct()
```

using

Instruction Activity Template

---

## Step 9

Verified generated XML.

Console

```
Depression Follow-up Entries = 2
```

---

## Step 10

Rendered HTML.

Verified section displayed correctly.

---

# Root Cause Analysis

## Actual Root Cause

The Treatment Plan generator never processed Depression Follow-up records.

Only standard Care Plan Instructions were exported.

---

## Contributing Factors

Instruction model lacked fields

* SNOMED
* Clinical Element Name
* GWID

Therefore structured coding could not be generated.

---

# Code Changes

---

## File

```
Instruction.java
```

### Added fields

```java
private String ClinicalElementName;
private String Gwid;
private String Snomed;
```

Added ResultSet mappings

```java
setClinicalElementName(...)
setSnomed(...)
setGwid(...)
```

---

## File

```
PlanofCareSectionHandlerUtil.java
```

---

### Narrative Section

Added

```java
planDto.getDepressionFollowupList()
```

Displayed

```
Depression Follow-up :
```

inside Treatment Plan.

---

### Date Collection

Included Depression Follow-up dates into

```
enDAte
```

to preserve chronological grouping.

---

### Structured XML

Added

```java
formDepressionFollowupAct()
```

Implementation

```java
CodeBean.CreateCode(
    instruction.getSnomed(),
    instruction.getClinicalElementName(),
    CodeSystem.SNOMED,
    "SNOMED CT",
    null
)
```

---

Generated

```xml
<entry>
    <act>

    </act>
</entry>
```

---

# SQL Analysis

## Investigation Query

```sql
SELECT
    clinical_elements_name AS instruction,
    to_mmddyyyy(encounter_date) AS date,
    clinical_elements_name,
    clinical_elements_snomed,
    patient_clinical_elements_gwid
FROM patient_clinical_elements
INNER JOIN encounter
ON encounter_id = patient_clinical_elements_encounterid
INNER JOIN clinical_elements
ON clinical_elements_gwid =
patient_clinical_elements.patient_clinical_elements_gwid
WHERE patient_clinical_elements_patientid = 6873
AND patient_clinical_elements_gwid IN
(
'0000409300000000063',
'0000409300000000065'
);
```

Purpose

Retrieve Depression Follow-up clinical elements.

---

Affected Tables

| Table                     | Purpose           |
| ------------------------- | ----------------- |
| patient_clinical_elements | Patient Follow-up |
| encounter                 | Encounter Date    |
| clinical_elements         | SNOMED mapping    |

---

Query Type

✔ Investigation Query

---

# Generated XML

Generated

```xml
<entry>
    <act classCode="ACT"
         moodCode="INT">

        <templateId
            root="2.16.840.1.113883.10.20.22.4.20"/>

        <code
            code="88848003"
            codeSystem="2.16.840.1.113883.6.96"
            codeSystemName="SNOMED CT"
            displayName="Follow-up for depression - adult">

            <originalText>
                <reference value="#Treatment1000"/>
            </originalText>

        </code>

        <text>
            Follow-up for depression - adult
        </text>

        <statusCode code="completed"/>

        <effectiveTime
            value="20260701"/>

    </act>
</entry>
```

---

# Validation Performed

Console

```
Depression Follow-up Count = 2
```

```
Depression Follow-up Entries = 2
```

```
Total Plan Of Care Entries = 2
```

---

HTML Validation

Treatment Plan rendered as

```
07/20/2026

Instruction :
Test Plan Note for Depression Follow-up

Depression Follow-up :
Follow-up for depression - adult

07/01/2026

Depression Follow-up :
Follow-up for depression - adult
```

---

XML Validation

Verified

```
<section>

templateId

2.16.840.1.113883.10.20.22.2.10
```

Generated entries

```
Instruction Activity

2.16.840.1.113883.10.20.22.4.20
```

---

# CCDA Structure Review

Verified structure

```xml
<component>

    <section>

        <templateId/>

        <code/>

        <title/>

        <text/>

        <entry>

            <act>

            </act>

        </entry>

    </section>

</component>
```

Structure is consistent with the existing implementation for the Treatment Plan section.

---

# Generated File Location

Generation path

```
/home/software/Documents/shared/log/CDAFiles/cda_Outbox/
```

Generated by

```
GenerateXDAAction
```

via

```java
CodeGeneratorUtility.CreateCCD(...)
```

Files generated

```
OUT-<timestamp>.cda
```

Readable XML

```
OUT-<timestamp>-Clinical-Summary.xml
```

---

# Investigation During Validation

The developer searched an older generated CCDA file:

```bash
grep -n "88848003" OUT-07-22-2026-05-49-11-497-Encounter-160383.cda
```

No results were returned.

Observation:

* The inspected file was generated **before** implementing the Depression Follow-up enhancement.
* Therefore, it did not contain the new XML entries.

Recommendation:

* Generate a **new CCDA** after deploying the changes.
* Verify the latest `.cda` file in the output directory instead of older generated files.

---

# Risks

* Incorrect SNOMED mapping may result in invalid coded entries.
* Duplicate Follow-up records could appear if duplicate clinical elements exist.
* Missing encounter dates would affect chronological ordering in the Treatment Plan narrative.

---

# Pending Items

* Validate the generated CCDA using an external CCDA validator (e.g., ONC/HL7 tooling) to ensure the new `Instruction Activity` entries conform to the implementation guide.
* Confirm with the product/domain team whether `Instruction Activity (2.16.840.1.113883.10.20.22.4.20)` is the preferred template for Depression Follow-up interventions or if a different template is required by customer specifications.
* Generate a fresh CCDA after deployment and verify that the latest `.cda` contains the new `88848003` entries.

---

# Lessons Learned

* Always inspect the **latest generated CCDA**, not previously generated files.
* Extend domain models (`Instruction`) before adding new CDA mappings to avoid missing coded fields.
* Validate both the **narrative (`<text>`)** and the **structured (`<entry>`)** sections, as one can appear correctly while the other may be missing.
* Add targeted console logging (record counts, generated entries, output file path) during development to simplify debugging.
* Preserve narrative-to-entry linkage by ensuring each `<reference value="#Treatment..."/>` corresponds to an element ID in the narrative section.

---
