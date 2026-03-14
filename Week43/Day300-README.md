# MIPS Flowsheet Not Loading (Calvary)

## Issue

**MIPS Flowsheet not loading for patient #009492**

Example URL generated:

```
https://glace-gwt.glaceemr.com/bdesktop/glaceemr.html?macraIntegration=true&patientId=9492&chartId=9807&encounterId=173939&providerId=&encdate=2025-04-14&userId=&userName=Test Doctor M.D.
```

### Problem in URL

```
providerId=
userId=
```

Both parameters are **empty**, so the **MIPS Flowsheet fails to load**.

---

# Root Cause Analysis

The MIPS flowsheet is opened using:

```
/home/software/git/glacelegacy/jsp/chart/patientdetails/EncounterDetail_2.jsp
```

Function:

```javascript
function openMIPSFlowSheet(patId, encounterId)
```

Provider is determined using:

```
billingdoctorid
```

If billing doctor is not found:

```
providerId=
userId=
```

which causes the MIPS flowsheet to fail.

---

# Possible Causes to Check

## 1️⃣ Session userId missing

Check session variables.

If `userId` is null → providerId becomes empty.

---

## 2️⃣ MACRA configuration disabled

Check:

```sql
SELECT initial_settings_option_value
FROM initial_settings
WHERE initial_settings_option_name ILIKE
'Do you want to enable MACRA desktop view?';
```

Expected result:

```
1
```

If `0` → MACRA flowsheet disabled.

---

## 3️⃣ Billing doctor issue (Main Problem)

Check billing doctor for DOS.

```sql
SELECT service_detail_bdoctorid
FROM service_detail
WHERE service_detail_patientid = 9492
AND service_detail_dos = '2025-04-14';
```

Result:

```
NULL
25
```

⚠️ First row is **NULL**.

But code uses:

```java
billingdoctorid = encIdsList.getJSONObject(0).get("billingdoctorid").toString();
```

This **always takes the first row**.

So:

```
billingdoctorid = ""
providerId =
userId =
```

This causes **MIPS flowsheet not loading**.

---

# Verify Encounter & Doctor Mapping

Check encounter details:

```sql
SELECT encounter_id, encounter_type
FROM encounter
WHERE encounter_id = 173939;
```

Result:

```
encounter_type = 1 (Office Visit)
```

---

Check doctor mapping:

```sql
SELECT 
    e.encounter_id,
    e.encounter_type,
    sd.service_detail_id,
    sd.service_detail_cptid,
    sd.service_detail_sdoctorid AS service_doctor_id,
    sdoc.emp_profile_fullname AS service_doctor_name,
    sd.service_detail_bdoctorid AS billing_doctor_id,
    bdoc.emp_profile_fullname AS billing_doctor_name
FROM encounter e
JOIN chart c 
    ON c.chart_id = e.encounter_chartid
JOIN service_detail sd
    ON sd.service_detail_patientid = c.chart_patientid
LEFT JOIN emp_profile sdoc
    ON sdoc.emp_profile_empid = sd.service_detail_sdoctorid
LEFT JOIN emp_profile bdoc
    ON bdoc.emp_profile_empid = sd.service_detail_bdoctorid
WHERE e.encounter_id = 173939
AND sd.service_detail_dos = '2025-04-14';
```

Result:

| service_detail_id | service doctor     | billing doctor   |
| ----------------- | ------------------ | ---------------- |
| 234766            | Automated          | NULL             |
| 234770            | Reagin Rhodes PA-C | Paul Pinson M.D. |

So the **correct billing doctor is 25 (Paul Pinson M.D.)**.

---

# Fix Applied

Update the NULL billing doctor.

```sql
UPDATE service_detail
SET service_detail_bdoctorid = 25
WHERE service_detail_patientid = 9492
AND service_detail_dos = '2025-04-14'
AND service_detail_bdoctorid IS NULL;
```

Verify:

```sql
SELECT service_detail_id, service_detail_bdoctorid
FROM service_detail
WHERE service_detail_patientid = 9492
AND service_detail_dos = '2025-04-14';
```

Result:

```
234766 | 25
234770 | 25
```

Now providerId becomes:

```
providerId=25
userId=25
```

MIPS flowsheet loads successfully.

---

# Safer Fix (Recommended)

Update only the affected row.

Step 1 – Find affected row

```sql
SELECT service_detail_id
FROM service_detail
WHERE service_detail_patientid = 9492
AND service_detail_dos = '2025-04-14'
AND service_detail_bdoctorid IS NULL;
```

Result:

```
234766
```

Step 2 – Update specific row

```sql
UPDATE service_detail
SET service_detail_bdoctorid = 25
WHERE service_detail_id = 234766;
```

---

# Revert Query (If Needed)

```sql
UPDATE service_detail
SET service_detail_bdoctorid = NULL
WHERE service_detail_id = 234766;
```

---

# Debug Query (Quick Doctor Check)

```sql
SELECT 
    sd.service_detail_id,
    e.encounter_type,
    sdoc.emp_profile_fullname AS service_doctor,
    bdoc.emp_profile_fullname AS billing_doctor
FROM service_detail sd
LEFT JOIN encounter e 
    ON e.encounter_chartid IN 
    (SELECT chart_id FROM chart WHERE chart_patientid = sd.service_detail_patientid)
LEFT JOIN emp_profile sdoc
    ON sdoc.emp_profile_empid = sd.service_detail_sdoctorid
LEFT JOIN emp_profile bdoc
    ON bdoc.emp_profile_empid = sd.service_detail_bdoctorid
WHERE sd.service_detail_patientid = 9492
AND sd.service_detail_dos = '2025-04-14';
```

---

# Key Learning

MIPS Flowsheet uses:

```
service_detail_bdoctorid
```

NOT

```
encounter_service_doctor
```

If the **first service_detail row has NULL billing doctor**, the system generates:

```
providerId=
```

and the MIPS flowsheet fails to load.

---

# Quick Troubleshooting Checklist

When **MIPS Flowsheet not loading**:

1️⃣ Check URL → providerId
2️⃣ Check MACRA config

```sql
SELECT initial_settings_option_value
FROM initial_settings
WHERE initial_settings_option_name ILIKE
'Do you want to enable MACRA desktop view?';
```

3️⃣ Check billing doctor

```sql
SELECT service_detail_bdoctorid
FROM service_detail
WHERE service_detail_patientid = ?
AND service_detail_dos = ?;
```

4️⃣ Fix NULL billing doctor if needed.

---
