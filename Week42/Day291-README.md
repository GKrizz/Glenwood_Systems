
#  QRDA III Not Respecting Visit-Type Filters (Glace EMR)

---

# 🎯 Objective

Fix mismatch where:

> ✅ UI Preview → shows **filtered (correct)** values
> ❌ QRDA III Export → shows **unfiltered (old)** values

---

# 🚨 Issue Summary

When user applies **Filter Report**:

* Exclude Hospital Visit
* Exclude ER Visit
* Exclude Nursing Home
* Exclude Assisted Living

### ✅ UI Behavior

* Denominator, Numerator, Rate → recalculated correctly

### ❌ QRDA III Behavior

* Still uses **unfiltered dataset**

---

# 🧠 Root Cause

## 🔍 Flow Analysis

```text
UI
 │
 │ Filter applied
 ▼
MIPSPerformanceReport.Action (mode=1)
 │
 ▼
getMIPSPerformanceRate()  ✅ filters applied
 │
 ▼
Dashboard UI  ✅ correct
```

---

### ❌ QRDA Flow

```text
Download QRDA III
 │
 ▼
MIPSPerformanceReport.Action (mode=6)
 │
 ▼
exportQRDAIII()
 │
 ▼
getMeasureRateReport() ❌ filters ignored
 │
 ▼
QRDA XML ❌ incorrect data
```

---

# 🔥 Actual Root Problem

Even though UI passes filters:

```http
filterExcludeHospitalVisit=true
filterExcludeERVisit=true
filterExcludeNursingHomeVisit=true
filterExcludeAssistedLivingVisit=true
```

👉 Inside QRDA:

```json
"excludeVisit": {
  "excludeERVisit": false,
  "excludeNursingHomeVisit": false,
  "excludeAssistedLivingVisit": false,
  "excludeHospitalVisit": false
}
```

❌ Filters are LOST before reaching service layer

---

# 📊 Why Only 2 Measures Affected

| Measure                     | Reason      |
| --------------------------- | ----------- |
| CMS68 (Current Medications) | Visit-based |
| CMS22 (BP Screening)        | Visit-based |

👉 These depend on **encounter type**

Other measures:

* Diagnosis-based
* Medication-based
  ➡️ Not impacted

---

# 🛠️ Fix Strategy

---

## 1️⃣ Ensure Filters Flow End-to-End

### ✔ Already done in UI

```javascript
params += "&filterExcludeHospitalVisit=" + filtersForQRDA['filterExcludeHospitalVisit']
       + "&filterExcludeERVisit=" + filtersForQRDA['filterExcludeERVisit']
       + "&filterExcludeNursingHomeVisit=" + filtersForQRDA['filterExcludeNursingHomeVisit']
       + "&filterExcludeAssistedLivingVisit=" + filtersForQRDA['filterExcludeAssistedLivingVisit'];
```

---

## 2️⃣ Fix in `MIPSPerformanceReportAction.java`

### ❌ Current Problem

Filters NOT mapped into request object

---

### ✅ Fix

```java
boolean excludeHospital = Boolean.parseBoolean(request.getParameter("filterExcludeHospitalVisit"));
boolean excludeER = Boolean.parseBoolean(request.getParameter("filterExcludeERVisit"));
boolean excludeNursing = Boolean.parseBoolean(request.getParameter("filterExcludeNursingHomeVisit"));
boolean excludeAssisted = Boolean.parseBoolean(request.getParameter("filterExcludeAssistedLivingVisit"));
```

---

### ✅ Pass into filterConditions

```java
Map<String, Object> excludeVisit = new HashMap<>();
excludeVisit.put("excludeHospitalVisit", excludeHospital);
excludeVisit.put("excludeERVisit", excludeER);
excludeVisit.put("excludeNursingHomeVisit", excludeNursing);
excludeVisit.put("excludeAssistedLivingVisit", excludeAssisted);

filterConditions.put("excludeVisit", excludeVisit);
```

---

## 3️⃣ Fix in `ExportQRDAServiceImpl.java`

### ✔ Ensure request object is used

```java
Map<String, Object> excludeVisit =
    (Map<String, Object>) request.get("filterConditions").get("excludeVisit");

boolean excludeHospital = (boolean) excludeVisit.get("excludeHospitalVisit");
boolean excludeER = (boolean) excludeVisit.get("excludeERVisit");
boolean excludeNursing = (boolean) excludeVisit.get("excludeNursingHomeVisit");
boolean excludeAssisted = (boolean) excludeVisit.get("excludeAssistedLivingVisit");
```

---

## 4️⃣ Fix in `getMeasureRateReport()`

👉 This is CRITICAL

### ❌ Current

```sql
-- No visit filtering
SELECT *
FROM encounter
```

---

### ✅ Updated SQL Logic

```sql
WHERE 1=1

-- Exclude Hospital
AND (
    :excludeHospitalVisit = false
    OR encounter_type NOT IN ('HOSPITAL')
)

-- Exclude ER
AND (
    :excludeERVisit = false
    OR encounter_type NOT IN ('ER')
)

-- Exclude Nursing Home
AND (
    :excludeNursingHomeVisit = false
    OR encounter_type NOT IN ('NURSING_HOME')
)

-- Exclude Assisted Living
AND (
    :excludeAssistedLivingVisit = false
    OR encounter_type NOT IN ('ASSISTED_LIVING')
)
```

---

## 5️⃣ Apply Same Logic in:

* `MeasureCategoryIIISectionHandler.java`
* Any method calling:

  ```java
  getMeasureRateReport()
  ```

---

# 🧪 Validation Steps

---

## ✅ Step 1: UI Check

Apply filters → confirm:

```text
Denominator reduced ✔
Rate increased ✔
```

---

## ✅ Step 2: Logs Check

```text
Exclude Hospital: true
Exclude ER: true
Exclude Nursing: true
Exclude Assisted: true
```

---

## ✅ Step 3: QRDA Output Check

Before fix:

```text
DENOM = 4466 ❌
```

After fix:

```text
DENOM = 404 ✅
```

---

## ✅ Step 4: Debug Log Confirmation

```text
Excluded Patients: [IDs]
Adjusted Denominator: correct
```

---

# 🔒 Important Design Principle

> **QRDA must always use SAME dataset as UI**

---

# ⚠️ Common Mistakes to Avoid

| Mistake                     | Impact              |
| --------------------------- | ------------------- |
| Filters only in UI          | ❌ mismatch          |
| Not passing to service      | ❌ ignored           |
| Hardcoded false flags       | ❌ always unfiltered |
| Filtering after aggregation | ❌ wrong counts      |

---

# 📌 Final Fix Summary

| Layer          | Fix                   |
| -------------- | --------------------- |
| UI             | ✅ already correct     |
| Controller     | 🔧 read params        |
| Request Object | 🔧 inject filters     |
| Service        | 🔧 apply filters      |
| SQL            | 🔧 enforce conditions |

---

# 🏁 Final Outcome

| Scenario       | Result       |
| -------------- | ------------ |
| UI Preview     | ✅ Correct    |
| QRDA III       | ✅ Matches UI |
| CMS Submission | ✅ Accurate   |

---

# 💡 One-Line Takeaway

> **If filters don’t reach SQL, they don’t exist.**

---

# 📎 Reference Logs (Before vs After)

### ❌ Before

```text
excludeVisit = false,false,false,false
DENOM = 12
```

### ✅ After

```text
excludeVisit = true,true,true,true
Excluded Patients = [376094]
Adjusted DENOM = 11
```

---
