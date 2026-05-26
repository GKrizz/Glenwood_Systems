
# Day 368 – Error From Springs Backend – hammad

# Issue Summary

A backend exception occurred while processing MIPS/QPP measure calculations.

The error was caused by a:

```text id="i3n9z0"
duplicate key constraint violation
```

during insertion into:

```text id="rqk5yy"
quality_measures_patient_entries
```

---

# What Happened?

While saving QPP/MIPS patient measure data, the Spring backend attempted to insert a new row into:

```text id="pq7f55"
quality_measures_patient_entries
```

However, a row with the same unique/composite key already existed.

Because of that:

* PostgreSQL rejected the insert
* Transaction failed
* Backend request errored out
* MIPS/QPP processing stopped for that request

---

# Error Flow

```text id="u6q5m9"
QPPPerformanceController.java
        │
        ▼
saveOrUpdatePatientQDMStatus()
        │
        ▼
MeasureCalcServiceImpl.saveMeasureDetails()
        │
        ▼
saveAndFlush()
        │
        ▼
PostgreSQL
        │
        ▼
Duplicate Key Constraint Violation
```

---

# Affected Component

## Controller

File:

```text id="dj8hkn"
QPPPerformanceController.java
```

Method:

```java id="qk5b7i"
saveOrUpdatePatientQDMStatus()
```

Reference:

```text id="dprxnn"
.java:236
```

---

## Service Layer

File:

```text id="8vhg3u"
MeasureCalcServiceImpl.java
```

Method:

```java id="7e0q7w"
saveMeasureDetails()
```

Reference:

```text id="ckh7vh"
line 320
```

---

# Root Cause

Inside:

```java id="sd44zy"
MeasureCalcServiceImpl.saveMeasureDetails()
```

the code performs:

```java id="8f0w6q"
saveAndFlush()
```

directly.

This results in:

```text id="0ogt74"
blind insert behavior
```

meaning:

* no pre-check is done
* existing row is not validated
* duplicate records are attempted

---

# Problematic Behavior

Current logic:

```text id="6r5yy0"
Always INSERT
```

Expected logic:

```text id="xxj6du"
CHECK existing row first
    ├── exists → UPDATE
    └── not exists → INSERT
```

---

# Why Duplicate Happened

A record already existed for the same:

* patient
* provider
* measure
* reporting year
* CMS/QDM combination

The second insert attempted to create another row with the same unique key.

PostgreSQL correctly rejected it.

---

# Likely Unique Constraint

The table probably contains a composite unique index similar to:

```sql id="n4qqo2"
(
  patient_id,
  provider_id,
  measure_id,
  reporting_year,
  npi,
  criteria
)
```

or equivalent QPP uniqueness columns.

---

# Technical Explanation

## Current Code Pattern

```java id="kjlwmf"
repository.saveAndFlush(entity);
```

This assumes:

```text id="mu2kcn"
record does not already exist
```

But in recalculation/reprocessing flows:

* same patient may be recalculated
* same measure may run multiple times
* retries/rebuilds occur
* concurrent requests may happen

Result:

```text id="m8dywy"
duplicate insert attempts
```

---

# Why saveAndFlush() Failed

`saveAndFlush()` immediately forces SQL execution.

So instead of batching:

```text id="p7a8vx"
Hibernate issued INSERT immediately
```

and PostgreSQL returned:

```text id="0mjlwm"
duplicate key violates unique constraint
```

---

# Correct Fix

# Option 1 – Check Before Insert (Recommended)

Before insert:

```java id="l4t9ou"
existingRecord = repository.findExisting(...);

if(existingRecord != null){
    update existingRecord;
}else{
    insert newRecord;
}
```

---

# Option 2 – Use Upsert Logic

Use:

```text id="w7f3lo"
INSERT ... ON CONFLICT DO UPDATE
```

(PostgreSQL UPSERT)

This is safer for concurrent processing.

---

# Option 3 – Replace saveAndFlush()

Instead of forcing immediate insert:

```java id="4q8sje"
save()
```

combined with proper entity identification/update handling.

But this alone does NOT solve duplicate logic.

---

# Recommended Repository Query

Example:

```java id="58t9pb"
findByPatientIdAndMeasureIdAndProviderIdAndReportingYear(...)
```

Then:

* update existing row
* avoid duplicate inserts

---

# Recommended Fix Flow

```text id="g7jlwm"
saveMeasureDetails()
    │
    ├── Check existing entry
    │
    ├── Exists?
    │      │
    │      ├── YES → update fields
    │      │
    │      └── NO → insert new row
    │
    └── save()
```

---

# Additional Risk

This issue may happen again during:

* recalculation jobs
* scheduler reruns
* concurrent provider processing
* retry mechanisms
* bulk rebuilds

---

# Recommended Improvements

| Improvement             | Purpose                      |
| ----------------------- | ---------------------------- |
| Add existence check     | Prevent duplicate inserts    |
| Use UPSERT              | Safer concurrency handling   |
| Add logging             | Easier debugging             |
| Add retry-safe logic    | Prevent repeat failures      |
| Add transaction tracing | Better root-cause visibility |

---

# Suggested Debug Logging

```java id="2q1n9w"
log.info("Saving measure entry: patient={}, measure={}, provider={}, year={}",
          patientId, measureId, providerId, reportingYear);
```

Before insert/update decision.

---

# Final Conclusion

The error occurred because:

```text id="vjlwmr"
MeasureCalcServiceImpl.saveMeasureDetails()
```

attempted a direct insert using:

```java id="g9a4kc"
saveAndFlush()
```

without verifying whether the record already existed.

As a result:

* duplicate composite key insertion was attempted
* PostgreSQL rejected the insert
* backend request failed

---

# Resolution Direction

## Recommended Fix

✔ Add pre-check before insert
✔ Update existing rows instead of blindly inserting
✔ Consider PostgreSQL UPSERT strategy
✔ Make measure calculation idempotent/retry-safe
✔ Improve logging around saveMeasureDetails()

---

# Summary

| Item                           | Status |
| ------------------------------ | ------ |
| Root Cause Identified          | YES    |
| Duplicate Constraint Confirmed | YES    |
| saveAndFlush Blind Insert      | YES    |
| Existing Record Check Missing  | YES    |
| Recommended Fix Available      | YES    |
| UPSERT Recommended             | YES    |
