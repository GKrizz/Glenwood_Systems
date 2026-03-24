
# 📘 README: CMS50v13 – Closing the Referral Loop (Electronic Referrals Fix)

---

# 🎯 Objective

Fix issue where:

> ❌ **Numerator = 0 (0.00%)**
> even though
> ✅ **DRC is using electronic referrals (Direct Messaging)**

---

# 🚨 Issue Summary

### Current Output

```text
IPP = 622  
DENOM = 622  
NUMER = 0  ❌  
RATE = 0.00%
```

### Expected Behavior

If DRC uses **electronic referrals**, then:

> ✔ These SHOULD count in numerator
> 👉 if consultant report is received electronically

---

# 🧠 Root Cause

## 🔴 Current Logic (Incorrect)

```sql
rd.referral_reviewed_on IS NOT NULL
AND rd.referral_reviewed_on > referral_date
```

👉 This assumes:

* Manual review workflow only

---

## 🟡 But DRC Uses:

```text
Electronic Referrals (Direct Messaging)
```

So closure happens via:

```text
referral_details_direct_message_id
```

NOT via:

* referral_reviewed_on
* referral_details_revdate

---

# 🔥 Actual Problem

Your numerator logic is:

```sql
WHERE rd.referral_reviewed_on IS NOT NULL
```

❌ This ignores:

* Direct messages
* Electronic consultant reports

---

# 📊 Measure Logic (CMS50v13)

Numerator requires:

> **Consultant Report received AFTER referral**

CQL:

```cql
exists ["Communication, Performed": "Consultant Report"]
```

👉 Key point:

✔ It does NOT require "reviewed"
✔ It requires "received"

---

# 🧾 Mapping to Your DB

| CQL Concept         | Your DB Field                        |
| ------------------- | ------------------------------------ |
| Consultant Report   | `referral_details_direct_message_id` |
| Received datetime   | (derived / message timestamp)        |
| Related to referral | `referral_details_refid`             |

---

# 🛠️ Fix Strategy

---

## 1️⃣ Update Numerator Logic

### ❌ OLD

```sql
rd.referral_reviewed_on IS NOT NULL
```

---

### ✅ NEW (Add Electronic Referral Support)

```sql
(
    -- Manual review
    rd.referral_reviewed_on IS NOT NULL
    AND rd.referral_reviewed_on::date > i.referral_date::date

    OR

    -- Electronic referral (Direct Messaging)
    rd.referral_details_direct_message_id IS NOT NULL
)
```

---

## 2️⃣ FULL FIXED NUMERATOR CTE

```sql
numerator AS (
    SELECT DISTINCT i.patient_id
    FROM ipp i
    JOIN referral_details rd 
        ON rd.referral_details_refid = i.referral_id
    WHERE
    (
        -- Manual closure
        (
            rd.referral_reviewed_on IS NOT NULL
            AND rd.referral_reviewed_on::date > i.referral_date::date
            AND rd.referral_reviewed_on BETWEEN DATE '2025-01-01' AND DATE '2025-12-31'
        )

        OR

        -- Electronic closure (DRC use case)
        (
            rd.referral_details_direct_message_id IS NOT NULL
        )
    )
)
```

---

## 3️⃣ Optional (Better Accuracy)

If you have message timestamp:

```sql
AND message_received_date > referral_date
```

---

## 4️⃣ Debug Query (IMPORTANT)

Check how many electronic referrals exist:

```sql
SELECT COUNT(*)
FROM referral_details
WHERE referral_details_direct_message_id IS NOT NULL
AND referral_details_isactive = 1
AND COALESCE(referral_order_on, referral_details_ord_on)
    BETWEEN DATE '2025-01-01' AND DATE '2025-10-31';
```

---

## 5️⃣ Validate Impact

```sql
SELECT
    COUNT(*) AS total_referrals,
    COUNT(referral_reviewed_on) AS manual_reviews,
    COUNT(referral_details_direct_message_id) AS electronic_referrals
FROM referral_details
WHERE referral_details_isactive = 1;
```

---

# 🧪 Expected Outcome After Fix

| Metric    | Before  | After    |
| --------- | ------- | -------- |
| NUMERATOR | 0 ❌     | > 0 ✅    |
| RATE      | 0.00% ❌ | Real % ✅ |

---

# ⚠️ Important Edge Cases

### ❗ DO NOT count:

* Direct referral created but no report received

### ✔ Only count if:

* direct_message_id exists **AND**
* it represents **incoming consultant report**

---

# 🔒 Recommended Enhancement

If possible, add:

```sql
AND rd.direct_message_type = 'CONSULT_REPORT'
```

---

# 📌 Key Takeaways

* Measure is about **RECEIVED report**, not just review
* Electronic workflows must be included
* Direct messaging = valid numerator evidence

---

# 🏁 Final Conclusion

> ❌ Your system assumes manual workflow
> ✅ But DRC uses electronic workflow

👉 Fix = include **direct_message_id**

---
