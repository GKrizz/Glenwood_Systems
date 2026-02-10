## eCQM 2026 – Unable to display eCQM data (Dr. Pinson & MIM)

**Case #241452 – CALVARY / Dr. Pinson / MIM**
**Reporting Year:** 2026
**Status:** ✅ Fixed

---

## 📌 Background

Starting **Feb 3–6, 2026**, multiple accounts (Dr. Pinson, MIM, others) reported that:

* eCQM section fails to load after patient checkout
* UI shows:
  ❌ **“Unable to show eCQM data”**
* Issue affected:

  * All checked-out patients
  * Multiple accounts
  * Reporting Year **2026 only**

This confirmed a **system-level backend issue**, not patient- or account-specific.

---

## 🔍 Symptoms Observed

### UI

* Generic error message
* No measure results displayed

### API Behavior

API was called successfully:

```
GET /QPPPerformance/getCQMStatusByPatient
```

* ✅ HTTP 200 OK
* ❌ Response payload incomplete / logically incorrect

Example problematic response:

```json
"data": {
  "accountId": "",
  "patientId": 0,
  "measureInfo": { ... },
  "measureStatus": {}
}
```

---

## 🧠 What This Meant Technically

* Measure metadata (`measureInfo`) loaded correctly
* Measure calculation (`measureStatus`) **did not execute**
* Patient/provider context was **lost or short-circuited**
* Backend logic exited silently without throwing errors

---

## 🧪 What We Checked (Step-by-Step)

### 1️⃣ Patient & Encounter Validation

Verified patient had valid encounters in 2026:

```sql
SELECT e.encounter_service_doctor, COUNT(*)
FROM encounter e
JOIN chart c ON c.chart_id = e.encounter_chartid
WHERE c.chart_patientid = 6727
AND DATE(e.encounter_date) BETWEEN '2026-01-01' AND '2026-12-31'
GROUP BY e.encounter_service_doctor;
```

✅ Encounters exist
✅ Encounter year = 2026

---

### 2️⃣ Provider Context Validation

Checked **service doctor vs billing doctor**:

* Encounter service doctor: **Reagin Rhodes PA-C**
* Billing doctor: **Paul Pinson, MD**

Confirmed via:

* `encounter`
* `service_detail`
* `emp_profile`

This is critical because **2026 eCQM uses billing-doctor–based logic**.

---

### 3️⃣ Measure Configuration Validation

Confirmed CMS measures mapped correctly:

```sql
SELECT
  ep.emp_profile_fullname,
  qmpm.quality_measures_provider_mapping_measure_id
FROM quality_measures_provider_mapping qmpm
JOIN emp_profile ep
  ON ep.emp_profile_empid = qmpm.quality_measures_provider_mapping_provider_id
WHERE qmpm.quality_measures_provider_mapping_provider_id = 25
AND qmpm.quality_measures_provider_mapping_reporting_year = 2026;
```

✅ CMS-130 (CMS-68) correctly configured
✅ Provider configuration intact

---

### 4️⃣ Measure Execution Comparison

Compared two cases:

| Case                        | Result                    |
| --------------------------- | ------------------------- |
| Valid internal test patient | `measureStatus` populated |
| Real CALVARY patient        | `measureStatus = {}`      |

This confirmed:

* Engine runs
* Logic breaks **only for specific conditions**

---

## 🚨 Root Cause

### ❌ Incorrect Negation Category Used in CMS-130 Logic

CMS-130 Denominator Exception requires:

> **Procedure / Intervention NOT performed**
> **WITH negationRationale = Medical Reason**

However, the backend code checked negation like this:


## 🧩 Problem Statement

* API response for `getCQMStatusByPatient` returned:

  * `numerator > 0`
  * `denominatorException = 0`
* Even when **Medical Reason negation** existed in QDM data
* This resulted in **incorrect scoring** for CMS-130
* Expected behavior per CMS spec:

  * Encounters with **Medical Reason** → **Denominator Exception**
  * Should **NOT** count toward Numerator

---

## 🔍 What We Checked

### 1️⃣ API Input & Context

* Patient ID, Encounter ID, Billing Doctor ID
* Reporting Year = **2026**
* Verified billing-doctor–based encounter selection

---

### 2️⃣ Measure Configuration (DB Validation)

**Query used:**

```sql
SELECT
  ep.emp_profile_fullname AS provider_name,
  mpc.macra_provider_configuration_reporting_year AS year,
  qmpm.quality_measures_provider_mapping_measure_id AS measure_id,
  md.title AS measure_name
FROM macra_provider_configuration mpc
JOIN quality_measures_provider_mapping qmpm
  ON qmpm.quality_measures_provider_mapping_provider_id =
     mpc.macra_provider_configuration_provider_id
 AND qmpm.quality_measures_provider_mapping_reporting_year =
     mpc.macra_provider_configuration_reporting_year
LEFT JOIN measure_details md
  ON md.measure_id = qmpm.quality_measures_provider_mapping_measure_id
LEFT JOIN emp_profile ep
  ON ep.emp_profile_empid =
     mpc.macra_provider_configuration_provider_id
WHERE mpc.macra_provider_configuration_provider_id = 25
  AND mpc.macra_provider_configuration_reporting_year = 2026;
```

✅ Confirmed CMS-130 is correctly mapped to the billing provider.

---

### 3️⃣ CMS-130 Spec Validation

From CMS-130 definition:

**Denominator Exception:**

```
Procedure, Not Performed
OR
Intervention, Not Performed
WITH negationRationale in "Medical Reason"
```

Key takeaway:

* **Medical Reason is a VALUESET**
* It applies to **Procedure / Intervention**
* It is **NOT** a standalone Attribute event

---

### 4️⃣ Valueset & Category Check

```sql
SELECT valueset_name, valueset_oid, qdm_category
FROM ecqm_specifications_2026
WHERE valueset_oid = '2.16.840.1.113883.3.526.3.1007';
```

Result:

```
valueset_name = Medical Reason
qdm_category  = Attribute
```

⚠ Important:

* Even though stored as `Attribute` in DB
* CMS logic applies it to **Procedure / Intervention negation**

---

## 🚨 Root Cause

### ❌ Incorrect Category Passed to Negation Utility

In CMS-130 rules, negation was checked like this:

```java
QDMUtilities.checkNegationForEvent(
    procedure,
    QDMCategory.ATTRIBUTE,   // ❌ WRONG
    "2.16.840.1.113883.3.526.3.1007",
    measure,
    encounterStart,
    encounterEnd
)
```

### What happened internally:

* Measure 130 **does not define an Attribute QDM category**
* `measure.getSpecification().getQdmCategory().get("Attribute")` → `null`
* Utility returned `false`
* Negation ignored
* Numerator incremented incorrectly
* Denominator Exception never triggered

This caused **measureStatus to appear valid but logically wrong**.



---

## 🛠 Utility Method (Final Version)

```java
public static boolean checkNegationForEvent(
        QDM qdm,
        String qdmCategoryString,
        String valueSetOID,
        EMeasure measure,
        Date startDate,
        Date endDate) {

    Category qdmCategory =
        measure.getSpecification().getQdmCategory().get(qdmCategoryString);

    if (qdmCategory == null) return false;

    List<Valueset> qdmValueSetList =
        qdmCategory.getValueSetByOID(valueSetOID);

    if (qdmValueSetList == null || qdmValueSetList.isEmpty()) return false;

    if (IntervalUtilities.isDuring(qdm.getStartDate(), startDate, endDate)
        || IntervalUtilities.compareWithoutTime(qdm.getStartDate(), startDate)) {

        Negation negation = qdm.getNegation();
        if (negation != null) {
            return checkCodeAvailablity(
                qdmValueSetList,
                negation.getCode(),
                negation.getCodeSystemOID() == null ? "" : negation.getCodeSystemOID()
            );
        }
    }
    return false;
}
```

