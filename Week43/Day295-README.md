
# QRDA Visit Exclusion Issue – CMS130 & CMS317

## Background

In QRDA Category III export, the **visit exclusion filter** (Hospital / ER / Nursing Home / Assisted Living) was not correctly applied for measures:

* CMS130
* CMS317

The UI correctly reflected the exclusion counts, but the **generated QRDA XML contained incorrect denominator values**.

Example mismatch:

| Source             | Denominator |
| ------------------ | ----------- |
| UI                 | 11          |
| Generated QRDA XML | 12          |

This created **data inconsistency between UI reports and QRDA submission files**.

---

# Root Cause

The QRDA export logic was using **simplified exclusion logic** that only reduced the denominator count.

File involved:

```
MeasureCategoryIIISectionHandler.java
```

Existing implementation:

```
bean.setDenominatorCount(newDenom);
```

This logic only updated:

```
Denominator
```

But did **not update**:

* Numerator
* Numerator Exclusion
* Denominator Exclusion
* Denominator Exception

However, the **UI calculation logic** recalculates all counts using patient-level data.

---

# Investigation Steps

### 1. Identified where QRDA XML was generated

Flow traced:

```
QRDAController
   ↓
ExportQRDAServiceImpl
   ↓
CodeGeneratorUtility.generateCDA()
   ↓
MeasureCategoryIIISectionHandler
   ↓
getNarativeBlock()
   ↓
getMeasureRateReport()
```

---

### 2. Compared UI calculation logic

UI used a different method:

```
getMeasureRateReportByNPI()
```

This method applied visit exclusions using two important functions:

```
getHospitalVisitCount()
getExcludeCount()
```

These methods recalculate **all measure counts**.

---

### 3. Identified missing logic in QRDA

QRDA implementation was only doing:

```
Denominator = Denominator - ExcludedPatients
```

But UI logic performs:

```
1. Identify excluded patients
2. Remove those patients from QualityMeasuresPatientEntries
3. Recalculate:
   - Denominator
   - Numerator
   - Numerator Exclusion
   - Denominator Exclusion
   - Denominator Exception
```

---

# Fix Implemented

QRDA export was updated to reuse the same logic used by UI.

Updated flow:

```
getHospitalVisitCount()
        ↓
getExcludeCount()
        ↓
update all counts
```

Updated fields:

```
Denominator
Numerator
Numerator Exclusion
Denominator Exclusion
Denominator Exception
```

---

# Additional Issue Fixed

While passing the visit exclusion filter from UI to backend, the following error occurred:

```
ClassCastException:
LinkedHashMap cannot be cast to VisitExclusionBean
```

Reason:

Spring converts request JSON to `LinkedHashMap` when stored inside `Map<String,Object>`.

Fix:

```
ObjectMapper mapper = new ObjectMapper();
excludeVisit = mapper.convertValue(obj, VisitExclusionBean.class);
```

---

# Final Result

After fix:

| Metric                | Before Fix | After Fix |
| --------------------- | ---------- | --------- |
| Denominator           | Incorrect  | Correct   |
| Numerator             | Incorrect  | Correct   |
| Denominator Exclusion | Incorrect  | Correct   |
| Denominator Exception | Incorrect  | Correct   |
| Numerator Exclusion   | Incorrect  | Correct   |

QRDA XML now matches UI calculation.

---

# Example Output

Correct XML:

```
<list>
<item>Initial Patient Population : 2</item>
<item>Performance Rate : 0.0 %</item>
<item>Denominator : 11</item>
<item>Numerator : 0</item>
<item>Denominator Exclusions : 3</item>
<item>Denominator Exceptions : 0</item>
<item>Numerator Exclusions : 0</item>
</list>
```

---

# Key Files Involved

```
QRDAController.java
ExportQRDAServiceImpl.java
CodeGeneratorUtility.java
MeasureCategoryIIISectionHandler.java
ExportQRDAModel.java
```

Important methods:

```
getMeasureRateReport()
getMeasureRateReportByNPI()
getHospitalVisitCount()
getExcludeCount()
```

---
