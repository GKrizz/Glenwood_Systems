# HbA1c CPT II Generation – Diabetes Diagnosis Validation

<img width="1984" height="1770" alt="mermaid-diagram" src="https://github.com/user-attachments/assets/6ea7da70-3f93-4cda-bbe1-bbaa1ab2ce52" />



## Overview

This document describes the investigation and implementation of diabetes diagnosis validation for automated HbA1c CPT II code generation in Glace EMR.

The primary objective was to ensure that HbA1c CPT II codes are generated only when the patient has a qualifying diabetes diagnosis.

The investigation covered:

- HbA1c laboratory result retrieval.
- Mapping HbA1c result values to CPT II codes.
- Diabetes diagnosis validation using the patient problem list and encounter assessments.
- Repository query implementation and possible duplication.
- Database investigation for a test patient and encounter.
- Cleanup of test service records and HbA1c laboratory data.
- Validation considerations and future maintenance recommendations.

**Related commit:** `#62660`

**Commit review reason:** Fix HbA1c CPT generation by validating diabetes diagnosis before generating CPT codes.

<img width="5492" height="2982" alt="mermaid-diagram (1)" src="https://github.com/user-attachments/assets/7fb7ebb9-5cdc-4569-ad11-26f70f40b251" />

---

## Background / Context

The application contains automated CPT code generation logic based on configured HEDIS/MIPS-related measures.

For HbA1c, the application reads laboratory result values and maps them to CPT II codes.

However, generating a CPT II code based only on the presence of an HbA1c result is insufficient for the intended business rule. The patient must also have a qualifying diabetes diagnosis.

Two diagnosis sources are considered:

1. **Problem List:** An active, unresolved diabetes diagnosis associated with the patient.
2. **Patient Assessments:** A diabetes diagnosis associated with the current patient encounter.

The implementation combines these checks to determine whether HbA1c CPT II generation is allowed.

### Business rule

Generate an HbA1c CPT II code only when:

- The HbA1c measure is configured.
- A qualifying diabetes diagnosis exists in either the problem list or the current encounter assessment.
- The HbA1c result is present and valid.

---

## Problem Statement

The HbA1c CPT generation logic needed an additional diagnosis validation condition.

The expected behavior was:

- If a valid HbA1c result exists and the patient has a qualifying diabetes diagnosis, generate the corresponding CPT II code.
- If the patient does not have a qualifying diabetes diagnosis, do not generate the HbA1c CPT II code.

The investigation also identified overlapping diabetes diagnosis repository methods. These methods appeared similar but had differences in their filtering conditions.

A separate debugging activity involved checking whether a newly entered HbA1c result generated the expected CPT II service record.

---

## Impact

### Functional impact

Without diagnosis validation, HbA1c CPT II generation could occur without confirming the required diabetes diagnosis.

Adding diagnosis validation helps prevent inappropriate CPT II code generation.

### Data impact

The investigation involved:

- `lab_entries`
- `lab_entries_parameter`
- `service_detail`
- `cpt`
- `problem_list`
- `patient_assessments`

Test data was inspected, and cleanup SQL was prepared for specific service and laboratory records.

The conversation does not establish whether all suggested cleanup transactions were committed. Verify the database before assuming cleanup is complete.

### Reporting impact

HbA1c CPT II codes may contribute to downstream clinical quality and billing workflows. The exact downstream reporting impact of this change was not independently verified during the investigation.

---

## Environment Details

| Item | Value |
|---|---|
| Database | PostgreSQL |
| Database name | `ciama_nextrelease` |
| Application | Glace EMR |
| Backend project | `glaceemr_master_new` |
| Primary implementation file | `ChargesServicesImpl.java` |
| Test patient ID | `376440` |
| Test chart ID | `376820` |
| Test encounter ID | `290063` |
| HbA1c lab detail ID | `261731` |
| HbA1c parameter ID | `1282127` |
| Related commit | `#62660` |
| Investigation date | November 4, 2026 |

---

## System Components Involved

### 1. ChargesServicesImpl.java

The relevant implementation is in:

`saveServicesBasedOnHedisMeasuresConfig(...)`

This method contains the measure-specific CPT generation logic.

The HbA1c branch checks whether measure ID `1` is configured and whether the HbA1c result is available.

It then validates the patient's diabetes diagnosis before generating and processing the CPT code.

### 2. ProblemListRepository

Responsible for checking whether the patient has an active, unresolved diabetes diagnosis in the problem list.

Relevant method:

`countActiveDiabetesDiagnosis(patientId)`

### 3. PatientAssessmentsRepository

Responsible for checking whether the current encounter contains a qualifying diabetes diagnosis.

Relevant method under discussion:

`countDiabetesDiagnosis(patientId, encounterId)`

A second method, `countCurrentDiabetesDiagnosisForPatient(...)`, was also introduced during the investigation. Its overlap with the existing method was identified and its removal was considered.

### 4. Database tables

| Table | Purpose |
|---|---|
| `problem_list` | Stores patient problem-list diagnoses |
| `patient_assessments` | Stores diagnoses associated with patient encounters |
| `lab_entries` | Stores laboratory test details |
| `lab_entries_parameter` | Stores laboratory result parameters |
| `service_detail` | Stores generated or manually entered service records |
| `cpt` | Stores CPT code descriptions and identifiers |

---

## Investigation Timeline

### Step 1 – Identify the HbA1c generation requirement

The HbA1c branch was reviewed in `saveServicesBasedOnHedisMeasuresConfig(...)`.

The required change was to validate diabetes diagnosis before generating the HbA1c CPT II code.

### Step 2 – Review diabetes diagnosis sources

Two diagnosis sources were identified:

- Active and unresolved diabetes diagnosis in `problem_list`.
- Diabetes diagnosis associated with the current encounter in `patient_assessments`.

The existing problem-list repository method was confirmed as the intended active diabetes diagnosis check.

### Step 3 – Review PatientAssessmentsRepository methods

Three relevant repository methods were reviewed:

- `countTobaccoDiagnosis(...)`
- `countDiabetesDiagnosis(...)`
- `countCurrentDiabetesDiagnosisForPatient(...)`

The two diabetes methods had overlapping patient and encounter conditions.

However, they differed in:

- `patient_assessments_isactive` filtering.
- ICD code matching patterns.

The decision to remove the newer method was discussed, with the recommendation to check all callers before deletion.

### Step 4 – Inspect HbA1c laboratory results

The test patient and encounter were used to inspect HbA1c laboratory data.

The result found was:

- HbA1c value: `7`
- LOINC code: `17856-6`
- Lab detail ID: `261731`
- Parameter ID: `1282127`
- Test status: `3`

### Step 5 – Inspect service records

The `service_detail` table was queried to identify generated CPT codes for patient `376440`.

The first query showed two `002` service records and three other service records.

The two `002` records were subsequently absent from the later output.

This indicates that the displayed service data changed between the two query results. The conversation does not establish exactly which deletion operation caused that change.

### Step 6 – Prepare laboratory data cleanup

The HbA1c parameter and parent laboratory detail were identified for cleanup.

The recommended deletion order was:

1. Delete the parameter record.
2. Delete the parent laboratory record only if no other parameters remain.
3. Verify both records.
4. Commit the transaction only after verification.

### Step 7 – Review implementation and commit

The repository query and HbA1c diagnosis validation were reviewed.

The commit review reason was prepared as:

`Fix HbA1c CPT generation by validating diabetes diagnosis before generating CPT codes.`

---

## Root Cause Analysis

### Primary issue

The HbA1c CPT generation workflow needed a diabetes diagnosis validation step before generating the corresponding CPT II code.

### Contributing factors

1. HbA1c result availability and diabetes diagnosis eligibility are separate conditions.
2. Diabetes diagnosis information can exist in more than one source.
3. The repository contained overlapping diabetes diagnosis methods with different filtering behavior.
4. The `patient_assessments_isactive` flag introduced an additional distinction between the two methods.

### Important clarification

The investigation did not establish that the repository methods were functionally identical.

The older method:

- Filters by patient ID.
- Filters by encounter ID.
- Matches E10, E11 and E13 using broad prefix matching.
- Does not filter by `patient_assessments_isactive`.

The newer method:

- Filters by patient ID.
- Filters by encounter ID.
- Requires `patient_assessments_isactive = FALSE`.
- Uses explicit code and decimal-code matching.

Therefore, removing one method changes the effective filtering behavior unless the business rule confirms that the additional active-state condition is unnecessary.

### Confirmed versus unconfirmed root cause

**Confirmed:** The HbA1c workflow needed diagnosis validation.

**Not established:** Whether the original production issue was caused by missing diagnosis validation alone, incorrect assessment active-state filtering, an encounter mismatch, or another service-generation condition.

---

## Detailed Technical Findings

### 1. HbA1c CPT II code mapping

The mapping discussed during the investigation was:

| HbA1c result | CPT II code |
|---|---|
| Less than 7.0 | `3044F` |
| 7.0 to less than 8.0 | `3051F` |
| 8.0 to 9.0 inclusive | `3052F` |
| Greater than 9.0 | `3046F` |

**Boundary conditions:**

- Exactly `7.0` maps to `3051F`.
- Exactly `8.0` maps to `3052F`.
- Exactly `9.0` maps to `3052F`.
- Values greater than `9.0` map to `3046F`.

The result value `7` found during testing should therefore map to `3051F`, assuming the mapping implementation follows these boundaries.

### 2. Diagnosis validation logic

The intended validation combines the problem-list and current encounter assessment checks.

```java
boolean hasActiveProblemListDiabetes =
    problemListRepository.countActiveDiabetesDiagnosis(patientId) > 0;

boolean hasCurrentEncounterDiabetes =
    patientAssessmentsRepository.countDiabetesDiagnosis(
        patientId, encounterId
    ) > 0;

boolean hasDiabetesDiagnosis =
    hasActiveProblemListDiabetes || hasCurrentEncounterDiabetes;
```

The HbA1c CPT generation should proceed only when `hasDiabetesDiagnosis` is true and the result is valid.

### 3. Result validation

The previously discussed result validation checks:

- Result is not `null`.
- Result is not empty.
- Result is not `-1`.

Trimming the result before validation is recommended to avoid problems caused by leading or trailing whitespace.

### 4. Important service-processing consideration

The investigation identified a potential risk with calling `processServiceChange(...)` using an empty CPT code.

That method may modify or delete an existing unsubmitted service, depending on its implementation.

Therefore:

- Call `processServiceChange(...)` only after the diagnosis and result validations pass.
- Do not pass an empty CPT code simply because the diagnosis condition failed.
- Confirm the intended behavior for previously generated CPT services when a diagnosis is subsequently removed.

### 5. Problem-list diagnosis query

The confirmed problem-list query checks:

- Patient ID.
- Active problem-list entry.
- Unresolved diagnosis, or unresolved state represented by `NULL`.
- Diabetes diagnosis code family E10, E11 or E13.

Recommended repository implementation:

```java
@Query("SELECT COUNT(pl) FROM ProblemList pl WHERE pl.problemListPatientId = :patientId AND pl.problemListIsactive = TRUE AND (pl.problemListIsresolved = FALSE OR pl.problemListIsresolved IS NULL) AND (UPPER(TRIM(pl.problemListDxCode)) IN ('E10', 'E11', 'E13') OR UPPER(TRIM(pl.problemListDxCode)) LIKE 'E10.%' OR UPPER(TRIM(pl.problemListDxCode)) LIKE 'E11.%' OR UPPER(TRIM(pl.problemListDxCode)) LIKE 'E13.%')")
long countActiveDiabetesDiagnosis(@Param("patientId") Integer patientId);
```

### 6. Patient assessment diagnosis query

The existing encounter-specific method is:

```java
@Query("""
    SELECT COUNT(pa)
    FROM PatientAssessments pa
    WHERE pa.patient_assessments_patientId = :patientId
      AND pa.patient_assessments_encounterid = :encounterId
      AND (
          UPPER(TRIM(pa.patient_assessments_dxcode)) LIKE 'E10%'
          OR UPPER(TRIM(pa.patient_assessments_dxcode)) LIKE 'E11%'
          OR UPPER(TRIM(pa.patient_assessments_dxcode)) LIKE 'E13%'
      )
""")
long countDiabetesDiagnosis(
    @Param("patientId") Integer patientId,
    @Param("encounterId") Integer encounterId
);
```

The newer method discussed was:

```java
@Query("""
    SELECT COUNT(pa)
    FROM PatientAssessments pa
    WHERE pa.patient_assessments_patientId = :patientId
      AND pa.patient_assessments_encounterid = :encounterId
      AND pa.patient_assessments_isactive = FALSE
      AND (
          UPPER(TRIM(pa.patient_assessments_dxcode)) IN ('E10', 'E11', 'E13')
          OR UPPER(TRIM(pa.patient_assessments_dxcode)) LIKE 'E10.%'
          OR UPPER(TRIM(pa.patient_assessments_dxcode)) LIKE 'E11.%'
          OR UPPER(TRIM(pa.patient_assessments_dxcode)) LIKE 'E13.%'
      )
""")
long countCurrentDiabetesDiagnosisForPatient(
    @Param("patientId") Integer patientId,
    @Param("encounterId") Integer encounterId
);
```

### 7. Repository method decision

The two methods are similar but not identical.

Before deleting `countCurrentDiabetesDiagnosisForPatient(...)`, search for all usages:

```bash
grep -R -n \
  "countDiabetesDiagnosis\|countCurrentDiabetesDiagnosisForPatient" \
  src/main/java
```

Recommended decision process:

1. Identify every caller of both methods.
2. Confirm whether `patient_assessments_isactive = FALSE` is required by the business rule.
3. Confirm the intended ICD code matching behavior.
4. Standardize on one method only if the same filtering semantics are acceptable to every caller.
5. Compile and test after removing the unused method.

The database column has a default value of `FALSE`, but that alone does not establish that the flag is unnecessary. Its application-level meaning and update behavior must be verified.

### 8. Tobacco diagnosis query

The tobacco diagnosis method is separate from diabetes diagnosis validation:

```java
@Query("""
    SELECT COUNT(pa)
    FROM PatientAssessments pa
    WHERE pa.patient_assessments_patientId = :patientId
      AND pa.patient_assessments_encounterid = :encounterId
      AND (
          UPPER(TRIM(pa.patient_assessments_dxcode)) IN ('F17.200', 'Z72.0')
      )
""")
long countTobaccoDiagnosis(
    @Param("patientId") Integer patientId,
    @Param("encounterId") Integer encounterId
);
```

This method should not be removed as part of the HbA1c diabetes query cleanup.

---

## SQL Analysis and Scripts

### Investigation Queries

#### 1. Retrieve HbA1c results for a patient chart and encounter

**Type:** Investigation query.

**Tables:** `lab_entries`, `lab_entries_parameter`.

**Purpose:** Identify HbA1c test details and their associated parameter values.

```sql
SELECT
    le.lab_entries_testdetail_id,
    le.lab_entries_test_desc,
    lep.lab_entries_parameter_id,
    lep.lab_entries_parameter_name,
    lep.lab_entries_parameter_code,
    lep.lab_entries_parameter_value,
    lep.lab_entries_parameter_date,
    le.lab_entries_test_status
FROM lab_entries le
JOIN lab_entries_parameter lep
    ON lep.lab_entries_parameter_testdetailid =
       le.lab_entries_testdetail_id
WHERE le.lab_entries_chartid = 376820
  AND le.lab_entries_encounter_id = 290063
  AND le.lab_entries_test_desc ILIKE 'hemoglobin%'
ORDER BY
    le.lab_entries_testdetail_id,
    lep.lab_entries_parameter_id;
```

**Observed result:**

| Field | Value |
|---|---|
| Lab detail ID | `261731` |
| Test description | Hemoglobin a1c |
| Parameter ID | `1282127` |
| Parameter name | HEMOGLOBIN A1c |
| Parameter code | `17856-6` |
| Parameter value | `7` |
| Parameter date | `2026-11-04 06:30:09` |
| Test status | `3` |

#### 2. Retrieve eligible HbA1c results for the reporting year

**Type:** Investigation and validation query.

**Purpose:** Inspect HbA1c results with test statuses other than 2 and 7 and performance dates from January 1, 2026.

```sql
SELECT
    le.lab_entries_test_desc,
    le.lab_entries_perf_on,
    le.lab_entries_modifiedon,
    lep.lab_entries_parameter_name,
    lep.lab_entries_parameter_code,
    lep.lab_entries_parameter_code_system,
    lep.lab_entries_parameter_value,
    lep.lab_entries_parameter_date,
    le.lab_entries_test_status
FROM lab_entries le
JOIN lab_entries_parameter lep
    ON lep.lab_entries_parameter_testdetailid =
       le.lab_entries_testdetail_id
WHERE le.lab_entries_chartid = 376820
  AND le.lab_entries_test_status NOT IN (2, 7)
  AND le.lab_entries_test_desc ILIKE 'hemoglobin%'
  AND le.lab_entries_perf_on::date >= '2026-01-01'
  AND le.lab_entries_encounter_id = 290063;
```

**Observed result:**

| Field | Value |
|---|---|
| Test description | Hemoglobin a1c |
| Performance date | `2026-11-04 06:30:09` |
| Modified timestamp | `2026-11-04 16:01:10.228785` |
| Parameter name | HEMOGLOBIN A1c |
| Parameter code | `17856-6` |
| Code system | `2.16.840.1.113883.6.1` |
| Result value | `7` |
| Parameter date | `2026-11-04 06:30:09` |
| Test status | `3` |

The code system is the LOINC OID.

#### 3. Inspect patient service records and CPT descriptions

**Type:** Investigation query.

**Tables:** `service_detail`, `cpt`.

**Purpose:** Identify service records and their corresponding CPT codes.

```sql
SELECT
    sd.service_detail_id,
    c.cpt_cptcode,
    c.cpt_description,
    sd.service_detail_dos,
    sd.service_detail_modifieddate
FROM service_detail sd
JOIN cpt c
    ON c.cpt_id = sd.service_detail_cptid
WHERE sd.service_detail_patientid = 376440
ORDER BY sd.service_detail_dos DESC;
```

The first observed result included:

| Service ID | CPT code | Description | Date of service |
|---|---|---|---|
| 378197 | 002 | OVERWRITE CPT CODE | 2026-11-04 |
| 378148 | 002 | OVERWRITE CPT CODE | 2026-10-21 |
| 378116 | 90748 | COMBIVAX, HEP B & HEMO FLU B | 2026-10-09 |
| 378094 | J3262 | INJECTION TOCILIZUMAB 1 MG | 2026-09-16 |
| 378025 | 99215 | High | 2026-08-28 |

A later query output no longer contained service IDs `378197` and `378148`.

The later output still included other historical service records, including `378116`, `378094`, `378025`, `378026`, `377788`, `377785` and `377748`.

Do not assume all these records are related to HbA1c.

#### 4. Inspect diabetes diagnoses for the test patient

**Type:** Investigation query.

**Tables:** `problem_list`, `patient_assessments`.

**Purpose:** Identify diabetes diagnosis records from both diagnosis sources, including O24 codes for investigation.

```sql
SELECT
    'problem_list' AS source_table,
    problem_list_dx_code AS diagnosis_code,
    problem_list_dx_descp AS diagnosis_description,
    problem_list_isactive AS is_active,
    problem_list_last_mod_on AS diagnosis_date
FROM problem_list
WHERE problem_list_patient_id = 376440
  AND problem_list_isactive = TRUE
  AND UPPER(TRIM(problem_list_dx_code)) ~
      '^(E10|E11|E13|O24)([.]|$)'

UNION ALL

SELECT
    'patient_assessments' AS source_table,
    patient_assessments_dxcode AS diagnosis_code,
    patient_assessments_dxdescription AS diagnosis_description,
    patient_assessments_isactive AS is_active,
    patient_assessments_encounterdate AS diagnosis_date
FROM patient_assessments
WHERE patient_assessments_patientid = 376440
  AND UPPER(TRIM(patient_assessments_dxcode)) ~
      '^(E10|E11|E13|O24)([.]|$)'

ORDER BY diagnosis_date DESC;
```

**Observed result:**

| Source | Diagnosis code | Description | Is active | Diagnosis date |
|---|---|---|---|---|
| patient_assessments | E10.65 | Type 1 diabetes mellitus with hyperglycemia | false | 2026-08-24 18:37:03-04 |

No matching active diabetes problem-list entry appeared in this query result.

**Important:** This query did not select `patient_assessments_encounterid`. Therefore, the result does not establish whether E10.65 belongs to the test encounter `290063`.

#### 5. Verify diabetes diagnosis encounter association

**Type:** Investigation and validation query.

**Purpose:** Determine whether the patient's E10 diagnosis belongs to the current encounter.

```sql
SELECT
    patient_assessments_id,
    patient_assessments_patientid,
    patient_assessments_encounterid,
    patient_assessments_dxcode,
    patient_assessments_dxdescription,
    patient_assessments_isactive,
    patient_assessments_status,
    patient_assessments_encounterdate
FROM patient_assessments
WHERE patient_assessments_patientid = 376440
  AND UPPER(TRIM(patient_assessments_dxcode)) ~
      '^(E10|E11|E13)([.]|$)'
ORDER BY patient_assessments_encounterdate DESC;
```

**Interpretation:**

- If E10.65 belongs to encounter `290063`, it should match the encounter-specific assessment query when the other query conditions are satisfied.
- If it belongs to another encounter, it should not qualify through the current encounter assessment check.
- The active-state behavior must be interpreted according to the application's diagnosis convention.

The output of this verification query was not provided in the conversation. The encounter association remains unconfirmed.

---

### Cleanup Queries

> **Warning:** The following are destructive SQL operations. Run them only in the intended environment, after verifying the exact record IDs and their relationships.

#### 6. Verify the two `002` service records

**Type:** Validation query before cleanup.

**Table:** `service_detail`.

```sql
SELECT
    service_detail_id,
    service_detail_patientid,
    service_detail_cptid,
    service_detail_dos
FROM service_detail
WHERE service_detail_id IN (378197, 378148)
  AND service_detail_patientid = 376440;
```

#### 7. Delete the two `002` service records

**Type:** Test-data cleanup.

**Table:** `service_detail`.

The records targeted were:

- Service ID `378197`
- Service ID `378148`

Recommended transaction:

```sql
BEGIN;

DELETE FROM service_detail
WHERE service_detail_id IN (378197, 378148)
  AND service_detail_patientid = 376440;

SELECT service_detail_id
FROM service_detail
WHERE service_detail_id IN (378197, 378148);

-- Commit only after verifying the result.
COMMIT;
```

If the verification shows unexpected records or the deletion was not intended, use `ROLLBACK` instead of `COMMIT`.

The later service query no longer showed these two IDs, but the exact execution history was not established.

#### 8. Verify the HbA1c lab detail and parameter records

**Type:** Validation query before cleanup.

**Tables:** `lab_entries`, `lab_entries_parameter`.

```sql
SELECT *
FROM lab_entries
WHERE lab_entries_testdetail_id = 261731;

SELECT *
FROM lab_entries_parameter
WHERE lab_entries_parameter_testdetailid = 261731;
```

#### 9. Delete the HbA1c parameter and parent lab detail

**Type:** Test-data cleanup.

**Tables:** `lab_entries_parameter`, `lab_entries`.

The intended records were:

- Lab detail ID `261731`
- Parameter ID `1282127`

Recommended transaction:

```sql
BEGIN;

-- Delete the HbA1c parameter.
DELETE FROM lab_entries_parameter
WHERE lab_entries_parameter_id = 1282127
  AND lab_entries_parameter_testdetailid = 261731;

-- Delete the parent only if no parameters remain.
DELETE FROM lab_entries
WHERE lab_entries_testdetail_id = 261731
  AND NOT EXISTS (
      SELECT 1
      FROM lab_entries_parameter
      WHERE lab_entries_parameter_testdetailid = 261731
  );

-- Verify the parameter is gone.
SELECT *
FROM lab_entries_parameter
WHERE lab_entries_parameter_testdetailid = 261731;

-- Verify the parent is gone.
SELECT *
FROM lab_entries
WHERE lab_entries_testdetail_id = 261731;

-- Commit only after checking both results.
COMMIT;
```

If anything unexpected appears, execute `ROLLBACK` instead.

**Data integrity considerations:**

- The parameter row is deleted before its parent.
- The parent lab record is deleted only if no other parameter records reference it.
- Foreign-key constraints and other dependent tables should be checked before executing cleanup in a shared or production database.
- This cleanup targets the specified HbA1c record only; it does not delete the patient's other laboratory results.

---

## SQL Performance and Indexing Considerations

The investigation queries use patient, chart, encounter, and parent-child identifiers.

Potentially useful indexes to verify include:

- `lab_entries.lab_entries_chartid`
- `lab_entries.lab_entries_encounter_id`
- `lab_entries.lab_entries_testdetail_id`
- `lab_entries_parameter.lab_entries_parameter_testdetailid`
- `lab_entries_parameter.lab_entries_parameter_id`
- `service_detail.service_detail_patientid`
- `service_detail.service_detail_id`
- `problem_list.problem_list_patient_id`
- `patient_assessments.patient_assessments_patientid`
- `patient_assessments.patient_assessments_encounterid`

These are index-review recommendations, not confirmation that the indexes are missing or need to be created.

The `UPPER(TRIM(...))` and `ILIKE` expressions may prevent ordinary indexes on the unmodified text column from being used efficiently in some query plans. If these checks become performance-sensitive, inspect execution plans before adding functional indexes.

Example:

```sql
EXPLAIN ANALYZE
SELECT COUNT(pa)
FROM patient_assessments pa
WHERE pa.patient_assessments_patientid = 376440
  AND pa.patient_assessments_encounterid = 290063
  AND (
      UPPER(TRIM(pa.patient_assessments_dxcode)) LIKE 'E10%'
      OR UPPER(TRIM(pa.patient_assessments_dxcode)) LIKE 'E11%'
      OR UPPER(TRIM(pa.patient_assessments_dxcode)) LIKE 'E13%'
  );
```

Do not add indexes without checking existing index definitions and query plans.

---

## Code Changes

### Affected module

`ChargesServicesImpl.java`

Method:

`saveServicesBasedOnHedisMeasuresConfig(...)`

### Logic change

The HbA1c branch was updated/discussed to validate diabetes diagnosis before generating the CPT II code.

Conceptual implementation:

```java
if (measureIdList.contains("1")) {

    boolean hasActiveProblemListDiabetes =
        problemListRepository.countActiveDiabetesDiagnosis(patientId) > 0;

    boolean hasCurrentEncounterDiabetes =
        patientAssessmentsRepository.countDiabetesDiagnosis(
            patientId, encounterId
        ) > 0;

    boolean hasDiabetesDiagnosis =
        hasActiveProblemListDiabetes || hasCurrentEncounterDiabetes;

    if (hasDiabetesDiagnosis
            && resultValue != null
            && !resultValue.trim().isEmpty()
            && !"-1".equals(resultValue.trim())) {

        hemoglobinCptCode =
            generateCptCodeForHemoglobin(resultValue.trim());

        processServiceChange(
            "hemoglobin",
            hemoglobinCptCode,
            existingServiceMap,
            newCptCodes,
            saveServiceBean,
            ignoredCptList
        );
    }
}
```

This is a reference representation of the discussed logic, not a verified copy of the final committed source file.

### Before and after behavior

| Condition | Before diagnosis validation | With diagnosis validation |
|---|---|---|
| Valid HbA1c result, diabetes diagnosis exists | CPT generation can proceed | CPT generation can proceed |
| Valid HbA1c result, no diabetes diagnosis | CPT generation could proceed | CPT generation is skipped |
| Null or empty result | Result validation prevents generation | Result validation prevents generation |
| Result `-1` | Invalid result | Invalid result |
| Diabetes diagnosis exists only in another encounter | Depends on existing logic | Should not qualify through current encounter assessment check |

### Important implementation details

- The problem-list check is patient-level.
- The assessment check is patient-and-encounter-level.
- The two checks are combined using logical OR.
- Result validation and diagnosis validation must both pass.
- `processServiceChange(...)` should not be invoked with an empty CPT code when diagnosis validation fails.

---

## Fixes and Workarounds

### Permanent fix

The intended permanent fix is to add diabetes diagnosis validation before HbA1c CPT II generation.

The diagnosis check uses:

1. Active and unresolved diabetes diagnosis from the problem list.
2. Diabetes diagnosis from the current patient encounter.

### Repository method cleanup

The two assessment methods overlap:

- `countDiabetesDiagnosis(...)`
- `countCurrentDiabetesDiagnosisForPatient(...)`

Before removing either method:

1. Search all callers.
2. Confirm the required active-state filtering behavior.
3. Confirm the required ICD matching behavior.
4. Retain the method that matches the intended business rule.
5. Compile and test after removal.

### Test-data cleanup

For repeatable testing:

1. Inspect existing HbA1c lab details.
2. Inspect the associated parameter rows.
3. Inspect generated CPT service records.
4. Delete only records created for the test, using their exact IDs.
5. Re-run the test with a controlled HbA1c result and diagnosis setup.

Do not delete historical patient records merely to simplify a test.

---

## Validation and Testing

### Confirmed database observations

The following were observed during the investigation:

| Validation | Observed outcome |
|---|---|
| HbA1c result lookup | One result found |
| HbA1c value | `7` |
| LOINC code | `17856-6` |
| Test status | `3` |
| Problem-list diabetes result | No matching active entry appeared in the supplied query output |
| Patient assessment diabetes result | E10.65 was found |
| Assessment encounter association | Not confirmed by the supplied output |
| Initial service lookup | Two `002` records and other historical services appeared |
| Later service lookup | The two `002` records were no longer displayed |

### Expected CPT mapping validation

For a result of `7`, the expected mapping is `3051F` based on the discussed threshold logic.

The supplied service query did not show a `3051F` record for this patient.

This does not by itself prove that CPT generation failed. Other conditions may have prevented generation, including:

- The assessment diagnosis belonging to a different encounter.
- The assessment active-state filter.
- The measure configuration condition.
- Result retrieval or processing conditions.
- Service filtering or persistence behavior.

### Recommended test matrix

| Test case | Problem list diagnosis | Current encounter diagnosis | HbA1c result | Expected behavior |
|---|---|---|---|---|
| 1 | Present | Absent | Valid | Generate CPT |
| 2 | Absent | Present | Valid | Generate CPT |
| 3 | Present | Present | Valid | Generate CPT |
| 4 | Absent | Absent | Valid | Do not generate CPT |
| 5 | Absent | Diagnosis in another encounter only | Valid | Do not generate through current encounter check |
| 6 | Present | Absent | Null | Do not generate CPT |
| 7 | Present | Absent | Empty | Do not generate CPT |
| 8 | Present | Absent | `-1` | Do not generate CPT |

### CPT threshold test matrix

| HbA1c result | Expected CPT II code |
|---:|---|
| 6.9 | `3044F` |
| 7.0 | `3051F` |
| 7.9 | `3051F` |
| 8.0 | `3052F` |
| 9.0 | `3052F` |
| 9.1 | `3046F` |

These are expected outcomes based on the mapping discussed in the investigation. They should be verified against the actual implementation.

### Recommended application logging

Add trace-level or appropriately controlled diagnostic logging for:

- Patient ID.
- Encounter ID.
- Whether the problem-list diagnosis check passed.
- Whether the encounter diagnosis check passed.
- Whether the combined diagnosis check passed.
- Original and normalized HbA1c result.
- Generated CPT code.
- Whether service processing was invoked.

Example:

```java
System.out.println(
    "[HBA1C-CPT-TRACE] patientId=" + patientId
    + ", encounterId=" + encounterId
    + ", activeProblemListDiabetes=" + hasActiveProblemListDiabetes
    + ", currentEncounterDiabetes=" + hasCurrentEncounterDiabetes
    + ", hasDiabetesDiagnosis=" + hasDiabetesDiagnosis
    + ", result=" + normalizedResult
    + ", cptCode=" + hemoglobinCptCode
);
```

Avoid logging unnecessary patient-identifiable information in production logs. Follow the application's logging and privacy standards.

---

## Risks and Side Effects

### 1. Diagnosis filtering differences

Removing `countCurrentDiabetesDiagnosisForPatient(...)` and using `countDiabetesDiagnosis(...)` removes the `isactive = FALSE` condition.

This may change which assessment records qualify.

### 2. ICD code matching differences

The existing method uses broad patterns such as:

```sql
LIKE 'E10%'
```

The newer method uses more restrictive patterns:

```sql
IN ('E10', 'E11', 'E13')
OR LIKE 'E10.%'
OR LIKE 'E11.%'
OR LIKE 'E13.%'
```

The choice must match the accepted diagnosis coding rules.

### 3. Problem-list and encounter diagnosis differences

The problem-list check is patient-level, while the assessment check is encounter-level.

This distinction is intentional in the discussed implementation and should be preserved unless the business rule changes.

### 4. Existing service records

Skipping CPT processing when a diagnosis is absent does not necessarily remove an existing CPT II service.

The behavior for previously generated or unsubmitted services must be defined separately.

### 5. Destructive SQL cleanup

Deleting parent laboratory records can affect associated data and audit history.

Always verify dependent records, foreign keys, and environment before cleanup.

### 6. Repository backward compatibility

Removing a repository method may break other service callers.

Search all usages before removing it.

### 7. O24 diagnosis codes

The investigation query included O24 codes, but the Java diagnosis methods discussed only checked E10, E11 and E13.

The supplied diabetes value set was noted to include O24 codes.

Whether O24 should qualify for this HbA1c CPT generation rule remains a business-rule question. Do not silently expand the Java query without confirmation.

---

## Pending Work

The following items remain open or require evidence:

1. Confirm whether `countCurrentDiabetesDiagnosisForPatient(...)` was removed from the final implementation.
2. Confirm all usages of `countDiabetesDiagnosis(...)`.
3. Confirm whether `patient_assessments_isactive = FALSE` is required for current encounter diagnosis eligibility.
4. Verify whether E10.65 belongs to encounter `290063`.
5. Confirm whether O24 diagnoses should qualify.
6. Verify the final implementation of `generateCptCodeForHemoglobin(...)`.
7. Confirm the exact behavior of `processServiceChange(...)` when diagnosis validation fails.
8. Confirm whether existing HbA1c CPT II services should be retained or removed when a qualifying diagnosis is no longer present.
9. Execute and document the full diagnosis/result test matrix.
10. Confirm whether the test laboratory cleanup transaction was committed.
11. Confirm whether the two `002` service records were deleted intentionally and successfully.
12. Review the final commit diff for `#62660`.

---

## Troubleshooting Guide

### Scenario 1: HbA1c result exists, but CPT II code is missing

Check in this order:

1. Confirm that the HbA1c measure ID is configured.
2. Confirm that the laboratory result is retrieved.
3. Confirm that the result is not null, empty or `-1`.
4. Check the problem-list diabetes diagnosis.
5. Check the current encounter assessment diagnosis.
6. Verify the assessment encounter ID.
7. Verify the assessment active-state behavior.
8. Check the HbA1c CPT mapping.
9. Check whether `processServiceChange(...)` was invoked.
10. Check whether the service was persisted or excluded by existing service-processing logic.

### Scenario 2: Diabetes diagnosis exists, but validation returns false

Check:

- Patient ID passed to the repository.
- Encounter ID passed to the repository.
- Diagnosis code stored in the database.
- Whether the code is E10, E11 or E13.
- Whether the code contains unexpected whitespace or formatting.
- Whether the diagnosis belongs to the current encounter.
- Whether the problem-list entry is active and unresolved.
- Whether the assessment active-state filter is applied.

### Scenario 3: Incorrect CPT II code generated

Check the result-to-code mapping boundaries.

For example:

- `7.0` should map to `3051F`, not `3044F`.
- `8.0` should map to `3052F`.
- `9.0` should map to `3052F`.
- `9.1` should map to `3046F`.

Also verify that the result is normalized before numeric comparison.

### Scenario 4: Existing CPT service remains after diagnosis removal

Check whether the implementation skips service processing when diagnosis validation fails.

Skipping generation does not necessarily delete an already existing service record.

Confirm the intended lifecycle behavior before adding deletion logic.

### Scenario 5: Duplicate diagnosis repository methods

Run:

```bash
grep -R -n \
  "countDiabetesDiagnosis\|countCurrentDiabetesDiagnosisForPatient" \
  src/main/java
```

Compare the callers and query semantics before consolidating methods.

---

## Lessons Learned

### 1. Separate clinical eligibility from result availability

A laboratory result alone should not be treated as proof that a patient qualifies for a diagnosis-dependent CPT generation rule.

Validate the required diagnosis independently.

### 2. Use the correct diagnosis scope

Patient-level problem-list diagnoses and encounter-level assessments have different scopes.

Ensure that the correct patient and encounter identifiers are passed into repository methods.

### 3. Similar repository queries may have different behavior

Queries that appear equivalent may differ in active-state filters, code matching, null handling, and encounter scope.

Review the full query conditions before deleting or consolidating methods.

### 4. Verify database relationships before cleanup

A laboratory result consists of parent and parameter records.

Delete child records first and delete the parent only when no child records remain.

### 5. Use targeted trace logs

A single log statement showing the result value is insufficient to explain why a CPT code was not generated.

Log the diagnosis checks and generated CPT code separately.

### 6. Test threshold boundaries

For numeric mappings, always test values exactly at each boundary, not just values comfortably inside a range.

### 7. Preserve existing service behavior

Avoid calling service-processing methods with empty or invalid CPT values unless their behavior is explicitly understood.

---

## Future Maintenance Recommendations

- Keep the HbA1c diagnosis validation logic centralized where practical.
- Document the business meaning of `patient_assessments_isactive`.
- Standardize ICD code matching only after confirming all callers.
- Add unit tests for diagnosis validation and CPT threshold boundaries.
- Add integration tests covering problem-list and encounter assessment diagnosis sources.
- Use structured application logging rather than permanent `System.out.println` statements.
- Review query plans before adding indexes.
- Keep test-data cleanup scripts targeted and reversible.
- Document the intended behavior for existing CPT services when a diagnosis is removed.
- Confirm the accepted diagnosis code set, including the treatment of O24 codes.

---

## Commit Information

| Field | Value |
|---|---|
| Commit | `#62660` |
| Review request | Kindly review this commit |
| Reason | Fix HbA1c CPT generation by validating diabetes diagnosis before generating CPT codes |

### Suggested review description

Fix HbA1c CPT generation by validating diabetes diagnosis before generating CPT codes.

### Review checklist

- [ ] Diagnosis validation is applied before HbA1c CPT generation.
- [ ] Active unresolved problem-list diabetes diagnosis is checked.
- [ ] Current encounter diabetes diagnosis is checked.
- [ ] HbA1c result validation is retained.
- [ ] CPT mapping boundaries are correct.
- [ ] No empty CPT code is passed to service processing.
- [ ] Repository method usages are checked before removing duplicate methods.
- [ ] No unrelated tobacco or other diagnosis logic is changed.
- [ ] Test scenarios are verified.
- [ ] No unintended database cleanup or service deletion occurs.

---

## Final Technical Summary

The HbA1c CPT II generation workflow requires both a valid HbA1c result and a qualifying diabetes diagnosis.

The discussed implementation checks two diagnosis sources:

1. Active and unresolved diabetes diagnosis in `problem_list`.
2. Diabetes diagnosis associated with the current encounter in `patient_assessments`.

The investigation found an HbA1c result of `7` for patient `376440`, chart `376820`, encounter `290063`. Based on the discussed mapping, the expected CPT II code is `3051F`.

An E10.65 diagnosis was found in `patient_assessments`, but its encounter association was not confirmed in the supplied query output. Therefore, the investigation did not conclusively establish why the CPT II service was absent.

The most important remaining verification is to confirm the diagnosis encounter association, validate the final repository query semantics, and test the complete generation flow.

**Primary maintenance principle:** Do not generate HbA1c CPT II codes based only on laboratory result availability. Validate the required diagnosis and preserve the correct patient and encounter scope before processing the service.



