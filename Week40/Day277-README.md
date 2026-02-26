# 📘 Day 277 — #241850

## Exclude Deceased Patients from MIPS Performance Report (CSM)

### Account: `csm`

### Measure Example: **CMS122 (HbA1c Poor Control – Inverse Measure)**

Patient: **28509**

---

# 🧠 ISSUE

Deceased patients were still appearing in the **MIPS Performance Report**, including being counted in the **numerator** for HbA1c (inverse measure).

This is incorrect per CMS specification.

---

# 📌 Measure Context (Inverse Logic)

For CMS122 (HbA1c Poor Control):

| HbA1c Result     | Measure Status |
| ---------------- | -------------- |
| **> 9%**         | ✅ MET          |
| **< 9%**         | ❌ NOT MET      |
| **No test done** | ✅ MET          |

⚠ This is an **inverse measure**, meaning "bad outcome = numerator".

However:

> 🚫 Deceased patients must NOT be included in IPP, denominator, or numerator.

---

# 🔎 Root Cause

The deceased exclusion rule was implemented incorrectly:

### ❌ Existing Logic (Incorrect)

```java
eval(IntervalUtilities.getIntervalByDays($patient.getDod(), request.getMeasurementPeriodEnd()) < 0)
```

This logic incorrectly evaluates patient status relative to measurement period.

---

# 🧪 Debug Findings

Using your debug rule:

```java
int diff = IntervalUtilities.getIntervalByDays($patient.getDod(), $end);
```

Meaning of `diff`:

| Value | Meaning                    |
| ----- | -------------------------- |
| > 0   | DOD BEFORE measurement end |
| = 0   | DOD same day               |
| < 0   | DOD AFTER measurement end  |

CMS rule says:

> Exclude if patient died **on or before measurement period end**

So condition must be:

```java
DOD != null AND DOD <= measurementPeriodEnd
```

---

# ✅ Correct Exclusion Logic

### ✔ Recommended Clean Version

```java
// EXCLUDE DECEASED PATIENTS
eval(
    $patient.getDod() != null &&
    IntervalUtilities.compareWithoutTime(
        $patient.getDod(),
        request.getMeasurementPeriodEnd()
    ) <= 0
)
```

---

### ✔ If using getIntervalByDays

```java
eval(
    $patient.getDod() != null &&
    IntervalUtilities.getIntervalByDays(
        $patient.getDod(),
        request.getMeasurementPeriodEnd()
    ) >= 0
)
```

---

# 📊 Why Current Logic Failed

Your rule:

```java
eval(IntervalUtilities.getIntervalByDays($patient.getDod(), request.getMeasurementPeriodEnd()) < 0)
```

This means:

👉 Only excludes if DOD is AFTER measurement period end
👉 Which is logically impossible
👉 So deceased patients were never excluded

---

# 🎯 Expected Behavior

If:

```
DOD = 2025-06-15
Measurement End = 2025-12-31
```

Then:

```
Patient died before MP end
→ Must be excluded
→ Remove from IPP
→ Remove from denominator
→ Remove from numerator
```

---

# 🧩 Recommended Placement in Rules

Deceased exclusion should be applied at:

### ✔ IPP level (preferred)

or

### ✔ Denominator level

Never allow deceased patient to pass IPP.

---

# 🛠 Final Production Rule Example

```java
rule "Exclude Deceased Patients"
when
    request: Request($patient: patient, $end: measurementPeriodEnd)
    eval(
        $patient.getDod() != null &&
        IntervalUtilities.compareWithoutTime(
            $patient.getDod(),
            $end
        ) <= 0
    )
then
    request.setInitialPopulation(false);
    request.setDenominator(false);
    request.setNumerator(false);

    System.out.println("❌ EXCLUDED — Patient deceased before MP end");
end
```

---

# 📈 Impact on MIPS Report

After fix:

| Section       | Result          |
| ------------- | --------------- |
| IPP           | Patient removed |
| Denominator   | Removed         |
| Numerator     | Removed         |
| Performance % | Corrected       |

---
