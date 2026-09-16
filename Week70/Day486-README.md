# README – MIPS Flowsheet Billing Doctor Issue

## Issue Overview

The MIPS Flowsheet was opening with the **Service Doctor** as the `providerId` instead of the **Billing Doctor**, even though a Billing Doctor was configured for the encounter.

This issue was observed in the **AGC account** for:

```text
Patient ID    : 13785
Chart ID      : 14120
Encounter ID  : 162370
Reporting Year: 2026
```

### Expected Behavior

When **MIPS Report By Billing Doctor** is enabled, the MIPS Flowsheet should use the Billing Doctor's ID as the `providerId`.

For this encounter:

```text
Service Doctor : 3308 - Charlene Shookoff M.D.
Billing Doctor : 17   - Robert Schramm M.D.
```

Therefore, the expected MIPS URL was:

```text
providerId=17
```

---

## Root Cause

The `service_detail` table contained the correct Billing Doctor:

```sql
SELECT service_detail_bdoctorid
FROM service_detail
WHERE service_detail_patientid = 13785
  AND service_detail_sdoctorid = 3308
  AND service_detail_dos = '2026-09-16';
```

Result:

```text
billingdoctorid
---------------
17
```

However, the account-level setting in the `initial_settings` table was:

```text
MIPS Report By Billing Doctor = 0
```

The `CurrentEncounter.jsp` code only retrieves the Billing Doctor when this configuration is enabled:

```java
if(MipsReportByBillingDr.equals("1")){
    // Retrieve billing doctor
}
```

Because the setting was `0`, the Billing Doctor lookup was skipped.

As a result:

```text
billingdoctorid = -1
```

The JavaScript then falls back to the Service Doctor:

```javascript
if(billingdoctorid == "-1"){
    providerId = serviceDoctorId;
}
```

Therefore, the generated MIPS URL contained:

```text
providerId=3308
```

instead of:

```text
providerId=17
```

---

## Fix Applied

The account configuration was updated in the `initial_settings` table.

### Before

```text
initial_settings_option_id    : 10312
initial_settings_option_name  : MIPS Report By Billing Doctor
initial_settings_option_value : 0
initial_settings_visible      : false
```

### After

```text
initial_settings_option_id    : 10312
initial_settings_option_name  : MIPS Report By Billing Doctor
initial_settings_option_value : 1
initial_settings_visible      : false
```

SQL used:

```sql
UPDATE initial_settings
SET initial_settings_option_value = '1'
WHERE initial_settings_option_id = 10312
  AND initial_settings_option_name = 'MIPS Report By Billing Doctor';
```

---

## Result

After enabling the configuration, the application executes the Billing Doctor lookup.

For encounter `162370`:

```text
Service Doctor  = 3308
Billing Doctor  = 17
```

The MIPS Flowsheet now uses:

```text
providerId=17
```

and displays:

```text
Service Doctor: Robert Schramm M.D.
```

This confirms that the Billing Doctor configuration is now being applied correctly.

---

## Comparison With Working Account

The Calvary account was also checked for comparison.

| Account          | Service Doctor | Billing Doctor | MIPS Report By Billing Doctor |
| ---------------- | -------------: | -------------: | ----------------------------: |
| Calvary          |             26 |             25 |                             1 |
| AGC - Before Fix |           3308 |             17 |                             0 |
| AGC - After Fix  |           3308 |             17 |                             1 |

Calvary was already configured with:

```text
MIPS Report By Billing Doctor = 1
```

which is why its MIPS URL was correctly using:

```text
providerId=25
```

---

## Conclusion

The issue was **not caused by missing Billing Doctor data** in `service_detail`.

The Billing Doctor was correctly stored as `17`. The issue was that the account-level **MIPS Report By Billing Doctor** configuration was disabled (`0`).

The configuration was changed to `1`, enabling the existing Billing Doctor lookup logic in `CurrentEncounter.jsp`. The MIPS Flowsheet now passes the Billing Doctor ID (`17`) as the `providerId`.

### Status

**Fixed and verified.**
