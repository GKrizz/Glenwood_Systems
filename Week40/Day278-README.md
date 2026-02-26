## 📘 CMS347v8 (Measure ID 438)

# Statin Therapy for the Prevention and Treatment of Cardiovascular Disease

### Day 278 — #242260 — MIPS Statin Use (Dr. George Maly)

---

# 🧠 What This Issue Is About

This is a **MIPS Quality Measure issue** for **Statin Therapy (CMS347v8)**.

Dr. George Maly noticed that in his **2025 MIPS report**:

> Patients who have documented **statin allergy / intolerance** are being marked as **“NOT MET” ❌**

This is incorrect.

---

# 🚨 Why This Is a Problem

### Current System Behavior

Statin-allergic patients → counted as:

```
NOT MET
```

Impact:

| Classification  | Result                    |
| --------------- | ------------------------- |
| NOT MET         | ❌ Lowers MIPS performance |
| DENOM EXCLUSION | ✅ No penalty              |

So the doctor’s performance score is unfairly reduced.

---

# ✅ What Should Happen (Per CMS Spec)

If a patient:

* Has **documented statin allergy**
* OR has **statin intolerance**
* OR has valid contraindication

👉 They must be classified as:

```
DENOMINATOR EXCLUSION
```

NOT “NOT MET”.

---

# 📋 Measure Logic Overview (CMS347v8)

This measure includes **4 denominator pathways**:

1. Clinical ASCVD
2. LDL ≥ 190 mg/dL
3. Diabetes (age 40–75)
4. 10-year ASCVD risk ≥ 20%

If patient qualifies in any pathway:

→ They must receive statin
→ OR have valid exclusion

---

# 🔎 Your Current Checks (Correct So Far)

### ✅ CHECK 1 — Age

You verified:

```sql
EXTRACT(YEAR FROM AGE('2025-01-01', dob))
```

All patients fall in eligible age groups (40–75 or ≥21 depending on pathway).

Good.

---

### ✅ CHECK 2 — Qualifying Encounter

You confirmed all patients have valid CPT encounters in 2025.

Good.

---

# 🎯 Now The Real Missing Piece

You must check:

> Are these “NOT MET” patients having statin allergy documented?

Because if YES:

They must be reclassified to **DENOM EXCLUSION**.

---

# 🧩 What You Need To Validate Next

---

## ✅ CHECK 3 — Statin Allergy / Intolerance

Check in:

* patient_allergies table
* problem_list
* patient_assessments
* medication intolerance table (if exists)

---

### Example SQL — Allergy Table

```sql
SELECT *
FROM patient_allergies
WHERE patient_id IN (3057,5502,6238,6336,28341,28058,7833,28821,27709,9939,
5306,26989,28796,25560,27241,26881)
AND (
    allergy_description ILIKE '%statin%'
    OR allergy_code IN (/* statin SNOMED codes */)
);
```

---

### Example SNOMED for Statin Allergy

Common codes:

* 294954000 — Statin allergy
* 416098002 — Drug intolerance

(Use actual value-set from CMS)

---

# 🛠 What Logic Must Be Added

Inside numerator evaluation:

### Current Logic (Wrong)

```java
IF (no statin prescribed)
    → NOT MET
```

---

### Correct Logic

```java
IF (no statin prescribed) {
    IF (statin allergy OR intolerance documented)
        → DENOMINATOR EXCLUSION
    ELSE
        → NOT MET
}
```

---

# 🧠 Why System Is Currently Wrong

Your engine likely checks only:

* Prescription present → MET
* Prescription absent → NOT MET

It does NOT check:

* Allergy value set before classifying.

So exclusion logic is missing or placed after numerator classification.

---

# 🔧 Correct Order of Evaluation

Correct evaluation order must be:

```
1️⃣ Check IPP
2️⃣ Check Denominator
3️⃣ Check Denominator Exclusions
4️⃣ Check Numerator
```

Currently system is doing:

```
1️⃣ IPP
2️⃣ Denominator
3️⃣ Numerator
4️⃣ (Missing exclusion check)
```

---

# 🏁 What Paddy Wants Your Team To Do

### 1️⃣ Review All 4 Statin Pathways

Ensure exclusion logic exists in all.

### 2️⃣ Identify NOT MET Patients

Cross-check them against statin allergy documentation.

### 3️⃣ Fix Classification

Change:

```
NOT MET → DENOM EXCLUSION
```

for valid allergy cases.

---

# 📊 Expected Outcome After Fix

| Patient               | Before    | After       |
| --------------------- | --------- | ----------- |
| Statin allergy pt     | NOT MET ❌ | EXCLUSION ✅ |
| No statin, no allergy | NOT MET ❌ | NOT MET ❌   |
| Statin prescribed     | MET ✅     | MET ✅       |

---

# 💡 Important CMS Compliance Note

Per CMS347v8 specification:

> Patients with documented statin intolerance or allergy must be excluded from denominator.

Failure to do this:

* Reduces physician MIPS score
* Can cause audit finding
* Is compliance risk

---

# 🧾 Final Summary

### Problem:

Statin allergy patients counted as NOT MET.

### Root Cause:

Missing Denominator Exclusion logic for allergy/intolerance.

### Required Fix:

Add exclusion check BEFORE numerator classification.

---

If you want, I can now:

* 🔍 Build a complete SQL to identify affected patients
* 🧠 Write correct Drools rule for statin exclusion
* 📊 Generate impact analysis query (before vs after score)

Tell me which one you need 👍
