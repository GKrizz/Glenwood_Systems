# eCQM 2026 CMS Import, QDM Fix & Datastore Deployment

## 📊 CMS Measure Delta Summary (2025 → 2026)

| Category                  | Count |
| ------------------------- | ----- |
| 🆕 New                    | 3     |
| 🗑️ Removed               | 1     |
| 🔄 Updated (version only) | 46    |

---

## 🆕 Newly Added CMS Measures (2026)

```sql
SELECT DISTINCT split_part(cms_id, 'v', 1) AS cms_measure
FROM ecqm_specifications_2026
WHERE split_part(cms_id, 'v', 1) NOT IN (
    SELECT DISTINCT split_part(cms_id, 'v', 1)
    FROM ecqm_specifications_2025
)
ORDER BY cms_measure;
```

**Result**

* CMS1154
* CMS1173
* CMS149

---

## 🗑️ Removed CMS Measures (2026)

```sql
SELECT DISTINCT split_part(cms_id, 'v', 1) AS cms_measure
FROM ecqm_specifications_2025
WHERE split_part(cms_id, 'v', 1) NOT IN (
    SELECT DISTINCT split_part(cms_id, 'v', 1)
    FROM ecqm_specifications_2026
)
ORDER BY cms_measure;
```

**Result**

* CMS249

---

## 🔄 Version-Updated CMS Measures

```sql
SELECT
    split_part(y25.cms_id, 'v', 1) AS cms_measure,
    y25.cms_id AS cms_2025,
    y26.cms_id AS cms_2026
FROM
    (SELECT DISTINCT cms_id FROM ecqm_specifications_2025) y25
JOIN
    (SELECT DISTINCT cms_id FROM ecqm_specifications_2026) y26
ON split_part(y25.cms_id, 'v', 1) = split_part(y26.cms_id, 'v', 1)
AND y25.cms_id <> y26.cms_id
ORDER BY cms_measure;
```

➡ **46 CMS measures updated by version only**

---

## 📋 Unified CMS Status (NEW / REMOVED / UPDATED)

```sql
WITH y25 AS (
    SELECT split_part(cms_id, 'v', 1) AS cms_measure,
           MAX(cms_id) AS cms_id_2025
    FROM ecqm_specifications_2025
    GROUP BY 1
),
y26 AS (
    SELECT split_part(cms_id, 'v', 1) AS cms_measure,
           MAX(cms_id) AS cms_id_2026
    FROM ecqm_specifications_2026
    GROUP BY 1
)
SELECT
    COALESCE(y25.cms_measure, y26.cms_measure) AS cms_measure,
    y25.cms_id_2025,
    y26.cms_id_2026,
    CASE
        WHEN y25.cms_measure IS NULL THEN 'NEW'
        WHEN y26.cms_measure IS NULL THEN 'REMOVED'
        WHEN y25.cms_id_2025 <> y26.cms_id_2026 THEN 'UPDATED'
        ELSE 'UNCHANGED'
    END AS status
FROM y25
FULL OUTER JOIN y26 USING (cms_measure)
ORDER BY status, cms_measure;
```

---

## 🧪 QDM Category Validation Summary

### 🟢 No Change Required

Measures imported cleanly with **no missing QDM categories**.

### 🟡 Change Required – Resolved Using Backup Mapping

Missing `qdm_category` fixed using `ecqm_specifications_2026_bkp`.

### 🔴 Change Required – No Backup / Previous Year

No backup or prior mapping existed → **correctly defaulted to `Attribute`**.

✔ All decisions validated and confirmed correct.

---

## 📥 CMS144v14 CSV Import & Initial Validation

#### Step 1: Copy CSV to Server

```bash
rsync -hvzr --ignore-existing -e 'ssh -p 24' CMS144v14.csv \
root@192.168.2.3:/home/software/Documents/value_set_2026/ecqm_import_2026
```

#### Step 2: Import CSV into Database

```sql
\copy ecqm_specifications_2026
FROM '/home/software/Documents/value_set_2026/ecqm_import_2026/CMS144v14.csv'
CSV HEADER;
```

**Result**

```
COPY 993
```

#### Step 3: Row Count Verification

```sql
SELECT cms_id, COUNT(*)
FROM ecqm_specifications_2026
GROUP BY cms_id;
```

```sql
SELECT cms_id, COUNT(*)
FROM ecqm_specifications_2026
WHERE cms_id = 'CMS144v14'
GROUP BY cms_id;
```

#### Step 4: NULL Validation (Column-Level)

```sql
SELECT
  COUNT(*) FILTER (WHERE nqf_number IS NULL)         AS nqf_number_null,
  COUNT(*) FILTER (WHERE valueset_name IS NULL)      AS valueset_name_null,
  COUNT(*) FILTER (WHERE valueset_oid IS NULL)       AS valueset_oid_null,
  COUNT(*) FILTER (WHERE qdm_category IS NULL)       AS qdm_category_null,
  COUNT(*) FILTER (WHERE definition_version IS NULL) AS definition_version_null,
  COUNT(*) FILTER (WHERE expansion_version IS NULL)  AS expansion_version_null,
  COUNT(*) FILTER (WHERE code IS NULL)               AS code_null,
  COUNT(*) FILTER (WHERE code_system IS NULL)        AS code_system_null,
  COUNT(*) FILTER (WHERE expansion_id IS NULL)       AS expansion_id_null
FROM ecqm_specifications_2026
WHERE cms_id = 'CMS144v14';
```


---

## 🛠️ Case Study — CMS144v14 QDM Fix

### 🔍 NULL QDM Check

```sql
SELECT COUNT(*)
FROM ecqm_specifications_2026
WHERE cms_id = 'CMS144v14'
  AND (qdm_category IS NULL OR qdm_category = '');
```

### 📌 Missing ValueSets

```sql
SELECT cms_id, valueset_oid, COUNT(*) AS missing_qdm_count
FROM ecqm_specifications_2026
WHERE cms_id = 'CMS144v14'
  AND (qdm_category IS NULL OR qdm_category = '')
GROUP BY cms_id, valueset_oid;
```

### 🔁 Backup Validation

```sql
SELECT DISTINCT b.cms_id, b.qdm_category, n.valueset_oid
FROM ecqm_specifications_2026 n
JOIN ecqm_specifications_2026_bkp b
  ON n.cms_id = b.cms_id
 AND n.valueset_oid = b.valueset_oid
WHERE n.cms_id = 'CMS144v14'
  AND n.qdm_category IS NULL;
```

---

## ✏️ QDM Fix Queries (FINAL)

### Step 1 — Manual Attribute Update

```sql
UPDATE ecqm_specifications_2026
SET qdm_category = 'Attribute'
WHERE cms_id = 'CMS144v14'
  AND valueset_oid IN (
      '2.16.840.1.113762.1.4.1206.49',
      '2.16.840.1.113762.1.4.1206.51',
      '2.16.840.1.113762.1.4.1206.53'
  )
  AND (qdm_category IS NULL OR qdm_category = '');
```

### Step 2 — Backup-Driven Update

```sql
UPDATE ecqm_specifications_2026 n
SET qdm_category = b.qdm_category
FROM ecqm_specifications_2026_bkp b
WHERE n.cms_id = b.cms_id
  AND n.valueset_oid = b.valueset_oid
  AND (n.qdm_category IS NULL OR n.qdm_category = '')
  AND b.qdm_category IS NOT NULL;
```

---

## ✅ Post-Fix Validation Queries

```sql
SELECT COUNT(*) FROM ecqm_specifications_2026
WHERE qdm_category IS NULL OR TRIM(qdm_category) = '';
```

```sql
SELECT COUNT(*) FROM ecqm_specifications_2026
WHERE code ~* 'E\+';
```

```sql
SELECT code_system, COUNT(*)
FROM ecqm_specifications_2026
GROUP BY code_system
ORDER BY COUNT(*) DESC;
```

---

## 🛡️ Production Fix & Rollback Plan

### 🟢 STEP 0 — Safety Check

```sql
SELECT COUNT(*) AS total_rows FROM ecqm_specifications_2026;
```

### 🟢 STEP 1 — Backup Table

```sql
DROP TABLE IF EXISTS ecqm_specifications_2026_fix_bkp;
CREATE TABLE ecqm_specifications_2026_fix_bkp AS
SELECT * FROM ecqm_specifications_2026;
```

### 🟢 STEP 2 — Drop Original

```sql
DROP TABLE ecqm_specifications_2026;
```

### 🟢 STEP 3 — Restore

```sql
ALTER TABLE ecqm_specifications_2026_fix_bkp
RENAME TO ecqm_specifications_2026;
```

---

## 🔁 Datastore Sync (55.1)

```sql
\copy (SELECT * FROM ecqm_specifications_2026)
TO ecqm_specifications_2026_datastore_bkp_2025_TO_2026_jan_29_2025.csv
WITH CSV HEADER;
```

📁 Store file in:

```
/Temp/Gobal
```

---

## 🚀 Final Step — DataGateway Refresh

```text
https://datagateway.glaceemr.com/DataGateway/eCQMServices/updateECQMSpecification?reportingYear=2026
```

✔ If issue occurs → **restore from backup table immediately**

---
