# Case #241377 — IssueQRDA III Issue Fix

**Date:** 03-Feb-2026
**Case #:** 241377
**Assigned To:** Gobala Krishnan
**Case Owner:** Althaf Pattan
**Module:** DIRECT
**Product:** GlacePremium

---

## Issue Summary

QRDA Category III (QRDA III) XML files were **failing CMS upload (CT validation)** due to **invalid program name** and **incorrect date formats**.

---

## Errors Observed

1. **Invalid Program Name**

   * Error: `pcf is not valid`
   * CMS accepts only values like:

     * `MIPS_INDIV`
     * `MIPS_GROUP`
     * `MIPS`
   * `PCF` is **not allowed** for QRDA III submissions.

2. **Invalid Date Format**

   * Dates found in XML: `20251230`, `12312024`
   * CMS accepted formats:

     * `YYYY-MM-DD`
     * `YYYY-MM-DDThh:mm:ss`
   * Reporting period mismatch observed:

     * Start Date incorrectly set as `2024-12-31`
     * Expected: `2025-01-01`
     * End Date: `2025-12-31`

---

## Root Cause

* Legacy code was:

  * Generating **PCF** as program name in QRDA III
  * Stripping date separators (`-`) using `replaceAll`, resulting in invalid formats
* Manual date handling caused reporting period inconsistencies

---

## Fix Implemented

### 1. Program Name Correction

* Removed usage of `PCF`
* Updated logic to generate valid CMS program names:

  * `MIPS_INDIV` for individual providers
  * `MIPS_GROUP` for group submissions

### 2. Date Handling Fix

* Removed manual string manipulation of dates
* Implemented proper date handling using `DateRangeBean`
* Ensured reporting period:

  * **Start Date:** `2025-01-01`
  * **End Date:** `2025-12-31`

---

## Files Updated

* `src/main/java/com/glenwood/glaceemr/server/application/services/QRDASections/MeasureCategoryIIISectionHandler.java`
* `src/main/java/com/glenwood/glaceemr/server/application/services/chart/MIPS/CDABasicElementFactory.java`

---

## Current Status

* Fix completed
* Changes requested to be applied **across all required accounts and versions**
* Awaiting validation / confirmation after deployment

---

## Notes

* Manual XML correction is no longer required after this fix
* Future QRDA III downloads should pass CMS CT validation

---
