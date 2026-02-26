
# 📘 CMS159v13 — Depression Remission at Twelve Months

# 🧠 Measure Summary (Plain English)

A patient qualifies if:

### ✅ Initial Population

* Age **≥ 12 years**
* Has **PHQ-9 score > 9**
* PHQ-9 recorded within **7 days of an eligible encounter**
* Encounter occurs within **Denominator Identification Period**

---

### ✅ Denominator Identification Period

```
Measurement Period Start = 2025-01-01

Denominator Period:
MP Start − 14 months → MP Start − 2 months

= 2023-11-01 → 2024-11-01
```

---

### ✅ Numerator Requirement

Patient must have:

```
Index PHQ > 9
AND
Follow-up PHQ < 5
between 10–14 months after index
(300–425 days)
```

---

# 🗂️ STEP-BY-STEP SQL LOGIC

---

# STEP 1 — Find Depression Encounters

```sql
WITH depression_encounters AS (

    SELECT DISTINCT
        sd.service_detail_id,
        sd.service_detail_patientid AS patient_id,
        sd.service_detail_dos AS encounter_date

    FROM service_detail sd
    JOIN cpt c
        ON c.cpt_id = sd.service_detail_cptid
    JOIN problem_list pl
        ON pl.problem_list_patient_id = sd.service_detail_patientid

    WHERE sd.service_detail_patientid = 11123

      AND c.cpt_cptcode IN (
        '90791','90792','90832','90834','90837','90839',
        '96156','96158','96159',
        '99202','99203','99204','99205','99211','99212','99213','99214','99215',
        '99384','99385','99386','99387','99394','99395','99396','99397',
        '99421','99422','99423','99441','99442','99443',
        'G0402','G0438','G0439'
      )

      AND pl.problem_list_dx_code IN (
        'F32.0','F32.1','F32.2','F32.3','F32.4','F32.5','F32.9',
        'F33.0','F33.1','F33.2','F33.3','F33.40','F33.41','F33.42','F33.9',
        'F34.1'
      )

      AND (pl.problem_list_resolved_date IS NULL
           OR pl.problem_list_resolved_date >= sd.service_detail_dos)

      AND sd.service_detail_dos BETWEEN DATE '2023-11-01' AND DATE '2024-11-01'
)
```

---

# STEP 2 — Extract PHQ-9 Assessments

```sql
, phq_assessments AS (

    SELECT
        patient_clinical_elements_patientid AS patient_id,
        patient_clinical_elements_created_on::date AS phq_date,
        patient_clinical_elements_value::int AS score

    FROM patient_clinical_elements

    WHERE patient_clinical_elements_gwid = '0000200200100183000'
      AND patient_clinical_elements_value ~ '^[0-9]+$'
)
```

---

# STEP 3 — Identify Index Event

```sql
, index_event AS (

    SELECT
        e.patient_id,
        e.encounter_date,
        p.phq_date,
        p.score,

        ROW_NUMBER() OVER (
            PARTITION BY e.patient_id
            ORDER BY p.phq_date
        ) AS rn

    FROM depression_encounters e
    JOIN phq_assessments p
        ON p.phq_date BETWEEN e.encounter_date - INTERVAL '7 days'
                           AND e.encounter_date
       AND p.score > 9
)

, first_index AS (
    SELECT *
    FROM index_event
    WHERE rn = 1
)
```

---

# STEP 4 — Age Eligibility Check

```sql
, age_check AS (

    SELECT
        i.*,
        p.patient_registration_dob,
        DATE_PART('year', AGE(i.phq_date, p.patient_registration_dob)) AS age

    FROM first_index i
    JOIN patient_registration p
        ON p.patient_registration_id = i.patient_id
)
```

---

# STEP 5 — Find Valid Follow-up PHQ

```sql
, followups AS (

    SELECT
        a.patient_id,
        a.phq_date AS index_date,
        pce.patient_clinical_elements_created_on::date AS followup_date,
        pce.patient_clinical_elements_value::int AS score

    FROM age_check a
    JOIN patient_clinical_elements pce
        ON pce.patient_clinical_elements_patientid = a.patient_id
       AND pce.patient_clinical_elements_gwid = '0000200200100183000'

    WHERE pce.patient_clinical_elements_created_on::date
          BETWEEN a.phq_date + INTERVAL '10 months'
              AND a.phq_date + INTERVAL '14 months'
)
```

---

# STEP 6 — Numerator Calculation

```sql
, numerator AS (

    SELECT
        patient_id,
        index_date,
        MAX(followup_date) AS last_followup,
        MIN(score) FILTER (WHERE score < 5) AS remission_score

    FROM followups
    GROUP BY patient_id, index_date
)

SELECT
    a.patient_id,
    a.phq_date AS index_date,
    a.age,

    CASE WHEN a.age >= 12 THEN 'DENOMINATOR'
         ELSE 'NOT ELIGIBLE'
    END AS denominator_status,

    n.last_followup,
    n.remission_score,

    CASE
        WHEN n.remission_score IS NOT NULL THEN 'NUMERATOR MET'
        ELSE 'NOT MET'
    END AS numerator_status

FROM age_check a
LEFT JOIN numerator n
    ON a.patient_id = n.patient_id;
```

---

# 📊 Example Result Interpretation

For Patient **11123**:

```
Index PHQ = 2025-10-17 (Score = 20)
Follow-up PHQ = 2026-02-03 (Score = 4)

Days between = 109 days
Required = 300–425 days

👉 Numerator NOT MET
```

---

# ⚠️ Common Implementation Mistakes

### ❌ Using wrong PHQ GWID

Must use:

```
0000200200100183000  (PHQ-9 total score)
```

---

### ❌ Ignoring Denominator Identification Period

Encounters must be strictly between:

```
MP Start − 14 months
MP Start − 2 months
```

---

### ❌ Not enforcing 7-day rule

PHQ must be:

```
BETWEEN encounter_date − 7 days AND encounter_date
```

---

### ❌ Using encounter date instead of PHQ date for numerator window

Numerator window is based on:

```
INDEX PHQ DATE (NOT encounter date)
```

---

# 🧩 Final Measure Flow

```
Encounter + Depression DX
        ↓
PHQ > 9 within 7 days
        ↓
INDEX EVENT
        ↓
Check Age ≥ 12
        ↓
Wait 10–14 months
        ↓
PHQ < 5 ?
        ↓
NUMERATOR MET
```

---
