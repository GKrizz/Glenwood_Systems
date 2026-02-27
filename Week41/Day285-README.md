# QRDA III Upload Issue – Date Format Fix (QPP Compliance)

## 📅 Date

**February 27, 2026**

---

# 🧩 Issue Summary

While uploading the **QRDA III file** to the **QPP (Quality Payment Program)** portal for **Dr. Nassir**, the upload was failing due to an invalid date format error.

---

# ❌ Root Cause

The generated QRDA XML contained dates in the following format:

```
YYYYMMDD
Example: 20250101
```

However, **CMS QPP requires** the date format to be:

```
YYYY-MM-DD
Example: 2025-01-01
```

Because of this mismatch, QPP validation rejected the QRDA III file.

---

# 📍 Where the Issue Occurred

The problem was identified in the QRDA XML elements:

* `<effectiveTime>`
* `<serviceEvent>`
* `<performer><time>`

Example of incorrect XML:

```xml
<effectiveTime>
   <low value="20250101"/>
   <high value="20251231"/>
</effectiveTime>
```

---

# 🔎 Investigation Findings

## Step 1 – Input from UI

Reporting dates were received in this format:

```
MM/dd/yyyy
Example: 01/01/2025
```

---

## Step 2 – Internal Conversion

Inside the system, the `DateRangeBean` automatically converted dates to:

```
yyyyMMdd
Example: 20250101
```

---

## Step 3 – XML Generation

The method:

```
formEffectiveTimeWithNull()
```

was directly inserting this value into XML without formatting, resulting in the incorrect format.

---

# ✅ Solution Implemented

A centralized fix was applied inside:

```
CDABasicElementFactory.java
Method: formEffectiveTimeWithNull()
```

---

## 🔧 Fix Details

A helper method was created to convert dates into QPP-compliant format:

```java
private String formatDateForQPP(String date) {
    if(date == null || date.trim().isEmpty())
        return date;

    if(date.matches("\\d{4}-\\d{2}-\\d{2}"))
        return date;

    if(date.matches("\\d{8}")) {
        return date.substring(0,4) + "-" +
               date.substring(4,6) + "-" +
               date.substring(6,8);
    }

    return date;
}
```

---

## 🛠 Code Change Applied

Before:

```java
ele.setValue(Lowdate);
ele1.setValue(Highdate);
```

After:

```java
ele.setValue(formatDateForQPP(Lowdate));
ele1.setValue(formatDateForQPP(Highdate));
```

---

# ✅ Result After Fix

Generated XML now correctly shows:

```xml
<effectiveTime>
   <low value="2025-01-01"/>
   <high value="2025-12-31"/>
</effectiveTime>
```

This format is fully compliant with CMS QPP requirements.

---
