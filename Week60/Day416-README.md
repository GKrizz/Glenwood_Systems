
# FHIR Allergy Export - RxNorm Code Mapping Investigation

---

# Overview

This document captures the investigation performed for a reported issue related to **FHIR Allergy (`AllergyIntolerance`) export**, where the customer requested that the **correct RxNorm code** be sent instead of the currently exported value.

The investigation focused on tracing the allergy data flow from the database through the backend services into the generated FHIR `AllergyIntolerance` resource.

> **Status:** Investigation completed.
> **Implementation:** Not yet performed.
> **Pending:** Business confirmation from Althaf regarding which database field should be used for the exported allergy code.

---

# Background / Context

The customer reported that the exported FHIR Allergy resource should contain the **correct RxNorm code**.

During investigation it was found that the application stores multiple allergy-related identifiers inside the `patient_allergies` table:

* Allergy Code
* RxNorm Code
* Code System OID

However, the current FHIR export uses the allergy code instead of the RxNorm code.

---

# Problem Statement

## Existing Behavior

The generated FHIR `AllergyIntolerance.code` uses

```
pat_allerg_allergy_code
```

as the coding value.

Example:

```json
"code": {
  "coding": [
    {
      "system": "http://www.nlm.nih.gov/research/umls/rxnorm",
      "code": "68"
    }
  ]
}
```

Although the coding system is RxNorm, the exported code is taken from `pat_allerg_allergy_code`.

---

## Expected Behavior

If the customer expects an RxNorm identifier, the exported coding should instead use

```
patallerg_rxnorm_code
```

Example:

```json
"code": {
  "coding": [
    {
      "system": "http://www.nlm.nih.gov/research/umls/rxnorm",
      "code": "723"
    }
  ]
}
```

---

# Impact

Incorrect code mapping may result in:

* Invalid RxNorm coding in exported FHIR resources
* Downstream interoperability issues
* Third-party systems receiving incorrect medication identifiers
* FHIR validation failures depending on consumer validation rules

---

# Environment Details

| Item         | Value                |
| ------------ | -------------------- |
| Module       | FHIR Allergy Export  |
| Resource     | AllergyIntolerance   |
| FHIR Version | R4                   |
| Backend      | glaceemr_backend_new |
| Language     | Java                 |
| ORM          | JPA/Hibernate        |

---

# System Components Involved

## Database

```
patient_allergies
```

Important columns

| Column                  | Purpose                        |
| ----------------------- | ------------------------------ |
| pat_allerg_allergy_code | Existing allergy code          |
| patallerg_rxnorm_code   | RxNorm code                    |
| pat_allerg_codesystem   | OID representing coding system |
| patallerg_concept_type  | RxNorm concept type            |

---

## Model

```
PatientAllergies.java
```

Contains

```java
@Column(name="patallerg_rxnorm_code")
private String patallergRxnormCode;

@Column(name="patallerg_concept_type")
private String patallergConceptType;
```

---

## Bean

```
PatientAllergiesBean.java
```

Contains

```java
private String patAllergyRxnormCode;
private String patAllergyConceptType;
```

with corresponding getters/setters.

---

## Service

```
AllergiesServiceImpl.java
```

Responsible for

* loading allergy records
* mapping database columns
* populating `PatientAllergiesBean`

---

## FHIR Export

```
FHIRAllergyServiceImpl.java
```

Responsible for building

```
FHIR AllergyIntolerance
```

---

## Utility

```
FHIRR4ServiceImpl.java
```

Responsible for

* CodeableConcept creation
* Coding creation
* Code System URL mapping

---

# Investigation Timeline

## Step 1

Customer reported:

> Update the correct RxNorm code in the exported allergy resource.

---

## Step 2

Reviewed

```
patient_allergies
```

Found both

```
pat_allerg_allergy_code
```

and

```
patallerg_rxnorm_code
```

---

## Step 3

Reviewed

```
PatientAllergies.java
```

Confirmed entity contains

```java
patallergRxnormCode
```

---

## Step 4

Reviewed

```
PatientAllergiesBean.java
```

Confirmed bean contains

```java
patAllergyRxnormCode
```

---

## Step 5

Reviewed

```
AllergiesServiceImpl
```

Confirmed

`selectAllergyColumns()`

selects

```java
root.get(PatientAllergies_.patallergRxnormCode)
```

and maps it to

```java
patientAllergiesBean.setPatAllergyRxnormCode(...)
```

Therefore RxNorm code is already available throughout the backend.

---

## Step 6

Reviewed

```
FHIRAllergyServiceImpl
```

Found

```java
allergyIntolerance.setCode(
    FHIRR4Service.setCodeableConcept(
        eachAllergy.getPatAllergAllergyCode(),
        FHIRR4Service.getCodeSystemURL(eachAllergy.getPatAllergCodeSystem()),
        eachAllergy.getPatAllergAllergicTo()
    )
);
```

Observation:

The export uses

```
pat_allerg_allergy_code
```

instead of

```
patallerg_rxnorm_code
```

---

## Step 7

Reviewed

```
FHIRR4ServiceImpl
```

### setCodeableConcept()

```java
public CodeableConcept setCodeableConcept(String code,
                                          String system,
                                          String display)
```

This method simply creates

```
Coding.code
Coding.system
Coding.display
```

No mapping logic exists here.

---

## Step 8

Reviewed

```
getCodeSystemURL()
```

RxNorm mapping

```java
2.16.840.1.113883.6.88
```

maps correctly to

```
http://www.nlm.nih.gov/research/umls/rxnorm
```

Therefore the coding **system** is already correct.

---

# Root Cause Analysis

## Root Cause

The FHIR export uses

```
pat_allerg_allergy_code
```

as the coding value regardless of the code system.

As a result,

```
system = RxNorm
```

but

```
code = Allergy Code
```

instead of

```
code = RxNorm Code
```

---

## Contributing Factors

The application stores multiple identifiers:

* Allergy Code
* RxNorm Code

The export logic always references

```
getPatAllergAllergyCode()
```

without checking whether the coding system is RxNorm.

---

# Detailed Technical Findings

## Data Flow

```
patient_allergies
        │
        ▼
PatientAllergies
        │
        ▼
PatientAllergiesBean
        │
        ▼
FHIRAllergyServiceImpl
        │
        ▼
FHIRR4Service.setCodeableConcept()
        │
        ▼
FHIR JSON
```

---

## Current Mapping

```
Database
    │
    ▼
pat_allerg_allergy_code
    │
    ▼
FHIR Coding.code
```

---

## Available Alternative

```
Database
    │
    ▼
patallerg_rxnorm_code
```

Already available but unused during export.

---

# SQL Analysis and Scripts

## Investigation Queries

No SQL queries were executed during this investigation.

The investigation was performed entirely through source code review.

---

# Code Analysis

## File

```
FHIRAllergyServiceImpl.java
```

Current implementation

```java
allergyIntolerance.setCode(
    FHIRR4Service.setCodeableConcept(
        eachAllergy.getPatAllergAllergyCode(),
        FHIRR4Service.getCodeSystemURL(eachAllergy.getPatAllergCodeSystem()),
        eachAllergy.getPatAllergAllergicTo()
    )
);
```

---

## Potential Change (Pending Business Confirmation)

Possible implementation

```java
String allergyCode = eachAllergy.getPatAllergAllergyCode();

if ("2.16.840.1.113883.6.88".equals(eachAllergy.getPatAllergCodeSystem())
        && HUtil.Nz(eachAllergy.getPatAllergyRxnormCode(), "").length() > 0) {

    allergyCode = eachAllergy.getPatAllergyRxnormCode();
}

allergyIntolerance.setCode(
    FHIRR4Service.setCodeableConcept(
        allergyCode,
        FHIRR4Service.getCodeSystemURL(eachAllergy.getPatAllergCodeSystem()),
        eachAllergy.getPatAllergAllergicTo()
    )
);
```

> **Note:** This change has **not** been implemented. It is a proposed solution pending business confirmation.

---

# Validation Strategy

Before making any code changes, validate the following values during debugging:

```java
eachAllergy.getPatAllergCodeSystem()

eachAllergy.getPatAllergAllergyCode()

eachAllergy.getPatAllergyRxnormCode()
```

Expected scenario:

| Property     | Example                |
| ------------ | ---------------------- |
| Code System  | 2.16.840.1.113883.6.88 |
| Allergy Code | 68                     |
| RxNorm Code  | 723                    |

If this occurs, the current implementation is exporting the wrong code for RxNorm.

---

# Risks and Side Effects

Changing the exported code without confirmation could affect:

* Existing FHIR consumers
* External integrations
* Interoperability with systems expecting the current allergy code
* Historical exports

---

# Pending Work

## Business Confirmation Required

The implementation should **not** be changed until the following question is answered:

> Should the FHIR `AllergyIntolerance.code` element use:
>
> * `pat_allerg_allergy_code`
>
> or
>
> * `patallerg_rxnorm_code`
>
> when the coding system is RxNorm (`2.16.840.1.113883.6.88`)?

---


# Lessons Learned

* The RxNorm code is already persisted in the database and propagated to the service layer.
* The issue is not related to database retrieval but to the mapping logic during FHIR resource generation.
* `FHIRR4ServiceImpl.setCodeableConcept()` only constructs the FHIR `CodeableConcept`; it does not determine which source field to use.
* `getCodeSystemURL()` correctly resolves the RxNorm OID (`2.16.840.1.113883.6.88`) to the RxNorm URI, so the system mapping is correct.
* Business requirements should be confirmed before modifying identifier mappings, as multiple code fields exist for the same allergy record and external integrations may rely on the current behavior.
