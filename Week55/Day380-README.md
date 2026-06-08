
# QRDA-I Reporting Period Date Format Issue

## RCA, Technical Analysis, Fix Documentation, and Deployment Reference

---

# Overview

This document captures the complete investigation, root cause analysis (RCA), code changes, SQL validation, deployment discussion, and operational handling for the QRDA-I submission failure issue related to incorrect reporting period date formatting inside generated QRDA files.

The issue impacted multiple customer accounts and caused QRDA-I import failures in external validation/import systems.

This document serves as:

* Technical RCA
* Implementation documentation
* Troubleshooting guide
* Deployment reference
* Support/onboarding knowledge base
* Long-term maintenance reference

---

# Background / Context

QRDA-I files generated from the GlaceEMR system were being submitted to external systems for MIPS/QPP reporting.

External validators/import systems rejected the QRDA-I files for specific customer TINs with the following error:

> "Reporting Period is not found!"

The external team identified that the reporting period dates inside the generated QRDA XML were incorrectly formatted.

---

# Problem Statement

## Reported Issue

QRDA-I submissions failed for the following practices:

* First Care Medical
* GI Physicians
* Midwest Infectious Disease
* Stay Home I Will

External validator rejected the files because reporting period dates were generated in this format:

```xml
<low value="2026-01-01"/>
<high value="2026-03-31"/>
```

However, the validator expected:

```xml
<low value="20260101"/>
<high value="20260331"/>
```

---

# Impact

## Business Impact

* QRDA-I submissions failed
* MIPS/QPP reporting blocked
* External import system rejected files
* Customer escalation marked as CRITICAL
* Multiple production accounts affected

## Technical Impact

* Generated QRDA XML became non-compliant
* Reporting period parsing failed in downstream systems
* Existing QPP-related date conversion logic unintentionally impacted QRDA generation

---

# Environment Details

## Affected Areas

| Component             | Description                   |
| --------------------- | ----------------------------- |
| QRDA-I Generation     | XML reporting file generation |
| MIPS/QPP Module       | Reporting subsystem           |
| CDA XML Builder       | EffectiveTime generation      |
| Backend Java Services | QRDA XML generation logic     |
| Stable Environment    | Customer-facing environment   |
| Beta Environment      | Validation/testing deployment |

---

# System Components Involved

## Java Classes

### Primary Modified File

```text
src/main/java/com/glenwood/glaceemr/server/application/services/chart/MIPS/CDABasicElementFactory.java
```

---

## Key Methods

### Modified Method

```java
formEffectiveTimeWithNull()
```

### Removed Helper Method

```java
formatDateForQPP()
```

---

## Additional Components Investigated

### Controller

```text
QPPPerformanceController.java
```

### Services

```text
MeasureCalcServiceImpl.java
MUPerformanceRateServiceImpl.java
```

---

# Investigation Timeline

## Phase 1 — External Failure Report

External support team reported:

> "Reporting Period is not found!"

QRDA-I imports failed for multiple TINs.

---

## Phase 2 — Screenshot Analysis

External screenshots showed:

### Incorrect Generated XML

```xml
<effectiveTime>
    <low value="2026-01-01"/>
    <high value="2026-03-31"/>
</effectiveTime>
```

### Expected XML

```xml
<effectiveTime>
    <low value="20260101"/>
    <high value="20260331"/>
</effectiveTime>
```

---

## Phase 3 — Code Investigation

Investigation identified that:

```java
formatDateForQPP()
```

was converting:

```text
20260101 -> 2026-01-01
```

inside:

```java
formEffectiveTimeWithNull()
```

This formatting was originally introduced for QPP upload compatibility.

---

## Phase 4 — Root Cause Confirmation

The QRDA validator required CDA-compliant compact dates:

```text
YYYYMMDD
```

but the system was generating:

```text
YYYY-MM-DD
```

---

## Phase 5 — Fix Implementation

The conversion logic was removed.

---

## Phase 6 — Validation

Generated QRDA files were revalidated.

Dates were confirmed to generate correctly:

```xml
<low value="20260101"/>
<high value="20260331"/>
```

---

## Phase 7 — Deployment Planning

Discussions with team lead confirmed:

* Beta deployment allowed
* Stable deployment approved
* Apply changes across all versions

---

# Root Cause Analysis

# Root Cause

The QRDA XML generation layer reused a QPP-specific date formatting conversion.

## Problematic Logic

```java
ele.setValue(formatDateForQPP(Lowdate));
```

The helper method converted:

```text
YYYYMMDD -> YYYY-MM-DD
```

This violated QRDA validator expectations.

---

# Contributing Factors

## 1. Shared Formatting Logic

QPP formatting logic was reused for QRDA generation.

## 2. Lack of Format Isolation

No separation existed between:

* QPP formatting
* QRDA formatting

## 3. Missing Validator-Level Testing

Generated QRDA files were not validated against strict external parser expectations after the formatting change.

---

# Detailed Technical Findings

# Original Problematic Code

```java
ele.setValue(formatDateForQPP(Lowdate));
```

and

```java
ele1.setValue(formatDateForQPP(Highdate));
```

---

# Helper Method

```java
private String formatDateForQPP(String date) throws Exception {
    if(date == null || date.trim().isEmpty())
        return date;

    if(date.matches("\\d{4}-\\d{2}-\\d{2}"))
        return date;

    if(date.matches("\\d{8}")){
        return date.substring(0,4)+"-"+date.substring(4,6)+"-"+date.substring(6,8);
    }

    return date;
}
```

---

# Final Fixed Code

```java
public IVLTS formEffectiveTimeWithNull(String Highdate,String Lowdate,String nullflavour,boolean isTimeReq)throws Exception{
    IVLTS element = new IVLTS();

    IVXBTS ele = new IVXBTS();

    if(Lowdate != null && !Lowdate.equals("") && !Lowdate.equals("-1") && !Lowdate.equals(" ")){
        ele = new IVXBTS();
        ele.setValue(Lowdate);
    }else{
        ele = new IVXBTS();
        ele.getNullFlavor().add(nullflavour);
    }

    JAXBElement<IVXBTS> elementcountry = (new ObjectFactory()).createIVLTSLow(ele);
    element.getRest().add(elementcountry);

    IVXBTS ele1 = new IVXBTS();

    if(Highdate != null && !Highdate.equals("") && !Highdate.equals("-1") && !Highdate.equals(" ")){
        ele1 = new IVXBTS();
        ele1.setValue(Highdate);
    }else{
        ele1 = new IVXBTS();
        ele1.getNullFlavor().add(nullflavour);
    }

    JAXBElement<IVXBTS> elementcountry1 = (new ObjectFactory()).createIVLTSHigh(ele1);
    element.getRest().add(elementcountry1);

    return element;
}
```

---

# Before vs After Behavior

| Scenario         | Before     | After    |
| ---------------- | ---------- | -------- |
| Input            | 20260101   | 20260101 |
| Generated XML    | 2026-01-01 | 20260101 |
| Validator Result | Failed     | Passed   |

---

# SQL Analysis and Scripts

# Investigation Queries

## Query 1 — Validate Provider Configuration

```sql
select *
from macra_provider_configuration
where macra_provider_configuration_provider_id in (1)
and macra_provider_configuration_reporting_year = 2025;
```

### Purpose

Verify whether provider configuration existed for reporting year 2025.

### Tables

* macra_provider_configuration

### Type

Investigation Query

---

## Query 2 — Validate Configured Measures

```sql
select *
from quality_measures_provider_mapping
where quality_measures_provider_mapping_provider_id = 1
and quality_measures_provider_mapping_reporting_year = 2025;
```

### Purpose

Verify whether quality measures were configured for the provider.

### Tables

* quality_measures_provider_mapping

### Type

Investigation Query

---

# Configuration Insert Scripts

## Provider Configuration Insert

```sql
INSERT INTO macra_provider_configuration (
  macra_provider_configuration_provider_id,
  macra_provider_configuration_reporting_year,
  macra_provider_configuration_reporting_start,
  macra_provider_configuration_reporting_end,
  macra_provider_configuration_reporting_method,
  macra_provider_configuration_report_type,
  macra_provider_configuration_hardship_exemption,
  macra_provider_configuration_aci_end,
  macra_provider_configuration_aci_start
) VALUES (
  2417,
  2025,
  '2025-01-01',
  '2025-12-31',
  2,
  2,
  'f',
  '2025-12-31',
  '2025-01-01'
);
```

### Purpose

Create MIPS configuration for provider.

### Type

Configuration Setup Script

---

## Measure Mapping Insert

```sql
INSERT INTO quality_measures_provider_mapping (
  quality_measures_provider_mapping_provider_id,
  quality_measures_provider_mapping_reporting_year,
  quality_measures_provider_mapping_measure_id
) VALUES (
  2417,
  '2025',
  '134'
);
```

### Purpose

Associate provider with quality measures.

### Type

Configuration Setup Script

---

# Application Flow Analysis

# Controller Flow

## File

```text
QPPPerformanceController.java
```

### Important Logic

```java
configuredMeasures = providerInfo.get(0).getMeasures();
```

### Failure Condition

```java
if(performanceObj.size()==0 && !isHEClaimed)
{
    emptyObj.setMessage("No Measure credits recorded for the reporting year "+reportingYear);
}
```

---

# Service Layer Investigation

## File

```text
MeasureCalcServiceImpl.java
```

### Important Service Calls

```java
getMeasureRateReportByNPI()
getMeasureRateReport()
getGroupPerformanceCount()
```

---

# Validation and Testing

# Validation Steps Performed

## 1. QRDA XML Validation

Verified generated XML now contains:

```xml
<low value="20260101"/>
<high value="20260331"/>
```

---

## 2. External Validator Compatibility

Confirmed generated format matches expected validator requirements.

---

## 3. Configuration Validation

Validated:

* Provider configuration exists
* Measure mappings exist
* Reporting year configured correctly

---

## 4. Environment Validation

Confirmed:

* Beta deployment acceptable
* Stable deployment approved

---

# Deployment Notes

# Commit Information

```text
Commit ID: 57265
```

---

# Deployment Decision

Approved by:

* Althaf

Instruction:

* Apply fix to all versions

---

# Deployment Plan

| Environment  | Status   |
| ------------ | -------- |
| Beta         | Approved |
| Stable       | Approved |
| All Versions | Approved |

---

# Risks and Side Effects

# Potential Risks

## 1. QPP Compatibility

Removing formatting conversion may impact previous QPP-specific expectations if any module depended on dashed format.

---

## 2. Shared Method Usage

`formEffectiveTimeWithNull()` may be reused elsewhere.

Requires regression validation.

---

## 3. XML Consumer Dependencies

Different external systems may expect different date formats.

---

# Recommendations

# Immediate Recommendations

## 1. Separate Formatting Utilities

Create dedicated:

* QRDA formatter
* QPP formatter

Avoid shared transformations.

---

## 2. Add Validator-Level Tests

Automated validation for:

* QRDA XML schema
* Reporting period format

---

## 3. Improve Logging

Add logs for generated reporting period values.

Example:

```java
logger.info("QRDA Reporting Period LowDate : {}", Lowdate);
```

---

# Pending Work

## Pending Validation

* Full regression validation for QPP upload flow
* Verify no downstream consumers require dashed format

---

## Suggested Future Enhancements

* Centralized date formatting utility
* Environment-based XML validation
* XML schema validation before export

---

# Lessons Learned

# Technical Lessons

## 1. Shared Utilities Can Cause Cross-Module Issues

A QPP formatting change unintentionally broke QRDA generation.

---

## 2. External Validators Are Strict

Small formatting differences can invalidate entire submissions.

---

## 3. XML Standards Must Be Preserved

QRDA expects:

```text
YYYYMMDD
```

not:

```text
YYYY-MM-DD
```

---

## 4. Regression Testing Is Critical

Any CDA/XML generation change requires:

* schema validation
* external validator testing
* downstream compatibility checks

---

# Troubleshooting Guide

# Symptom

```text
Reporting Period is not found!
```

---

# Verify Generated XML

Check:

```xml
<effectiveTime>
```

values.

---

# Correct Format

```xml
<low value="20260101"/>
<high value="20260331"/>
```

---

# Incorrect Format

```xml
<low value="2026-01-01"/>
```

---

# Files to Inspect

```text
CDABasicElementFactory.java
```

---

# Key Method

```java
formEffectiveTimeWithNull()
```

---

# Check For

```java
formatDateForQPP()
```

usage.

---

# Final Outcome

The QRDA reporting period issue was successfully identified and fixed by removing QPP-specific date conversion logic from QRDA effective time generation.

The generated QRDA XML now produces reporting dates in validator-compliant format:

```text
YYYYMMDD
```

The fix was approved for deployment across all versions.
