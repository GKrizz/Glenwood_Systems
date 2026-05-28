
For **CMS22v13 – Preventive Care and Screening: Screening for High Blood Pressure and Follow-Up Documented**, the **Numerator** means:

> The patient encounter must have:

1. **Blood Pressure recorded**
2. AND the **correct follow-up action documented based on BP range**

---

# Step-by-Step Numerator Logic

The measure checks the **LAST BP during the encounter**:

* **Systolic BP** (top number)
* **Diastolic BP** (bottom number)

Both are mandatory.

---

# CASE 1 — Normal BP

## Condition

* Systolic < 120
* AND Diastolic < 80

Example:

* 118 / 76

## Numerator Result

✅ Numerator Met

## Follow-up needed?

❌ No follow-up required

---

# CASE 2 — Elevated BP

## Condition

* Systolic = 120–129
* AND Diastolic < 80

Example:

* 125 / 78

---

## Then ONE of these must exist

### Option A

Follow-up within 6 months
AND
Lifestyle intervention

Examples:

* Weight loss counseling
* Diet counseling
* Exercise counseling
* Alcohol counseling
* Lifestyle recommendation

---

### Option B

Referral to PCP / alternate provider

---

## Result

✅ Numerator Met only if:

* BP present
* AND intervention documented

Otherwise:

❌ Numerator Not Met

---

# CASE 3 — First Hypertensive Reading

## Condition

Current encounter:

* Systolic >= 130
  OR
* Diastolic >= 80

BUT

No previous hypertensive BP in prior 12 months.

Example:

* 135 / 82
* Previous year normal

---

## Required

### Either

Referral to PCP

OR

### BOTH

* Follow-up within 4 weeks
* Lifestyle intervention

---

## Result

If intervention missing:

❌ Numerator Not Met

---

# CASE 4 — Second Hypertensive Reading (130–139 / 80–89)

## Condition

Current BP:

* Systolic 130–139
  OR
* Diastolic 80–89

AND

Patient already had hypertensive BP within last 12 months.

---

## Required

### Either

Referral to provider

OR

### ALL THREE

1. Follow-up within 6 months
2. Lifestyle intervention
3. Lab test or ECG order

Examples:

* CBC
* BMP
* ECG
* Hypertension lab panel

---

## Result

Missing any required intervention:

❌ Numerator Not Met

---

# CASE 5 — Severe Hypertension (>=140 / >=90)

## Condition

Current BP:

* Systolic >= 140
  OR
* Diastolic >= 90

AND

Previous hypertensive BP exists within 12 months.

---

## Required

### Either

Referral to provider

OR

### ALL FOUR

1. Follow-up within 4 weeks
2. Lifestyle intervention
3. Hypertension medication
4. Lab test or ECG

---

# Important Core Rule

## BOTH BP values are mandatory

The measure explicitly says:

> Both systolic and diastolic BP measurements are required.

If one is missing:

❌ Numerator fails

---

# In Your System

You are currently validating numerator using CPT/G-codes like:

## Systolic Codes

| Code  | Meaning          |
| ----- | ---------------- |
| G8752 | Systolic <140    |
| G8753 | Systolic >=140   |
| 3074F | Systolic <130    |
| 3075F | Systolic 130–139 |
| 3077F | Systolic >=140   |

---

## Diastolic Codes

| Code  | Meaning         |
| ----- | --------------- |
| G8754 | Diastolic <90   |
| G8755 | Diastolic >=90  |
| 3078F | Diastolic <80   |
| 3079F | Diastolic 80–89 |
| 3080F | Diastolic >=90  |

---

# Your Patient 1138

You found:

```text id="if9brj"
G8752 → systolic present
G8754 → diastolic present
```

Meaning:

* Systolic BP recorded
* Diastolic BP recorded

So:

✅ BP screening exists

But CMS22 numerator also depends on:

* actual BP range
* follow-up documentation/interventions

---

# Simple Numerator Decision Flow

```text id="v6x9fa"
BP present?
   NO  → Numerator FAIL

   YES
      ↓

Normal BP?
   YES → Numerator PASS

Elevated / Hypertensive?
   YES
      ↓

Correct follow-up documented?
   YES → Numerator PASS
   NO  → Numerator FAIL
```

---

# Most Important Tables for You

| Purpose                 | Likely Table                       |
| ----------------------- | ---------------------------------- |
| BP CPT/G-codes          | service_detail                     |
| Actual vitals           | patient_vitals                     |
| Diagnoses               | patient_assessments / problem_list |
| Lab orders              | lab_entries                        |
| Follow-up interventions | service_detail / plan tables       |
| Medications             | medication tables                  |

---

# In Your Current Implementation

Looks like numerator logic is mostly driven through:

* CPT/G-codes in `service_detail`
* not directly from `patient_vitals`

Example:

```text id="97q7bx"
G8752
G8754
```

already indicates BP captured.

So your current validation approach is correct.
