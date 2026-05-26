
# Day 352 – Errors on ADT Inbound for Glenwood

**Account:** Konni
**Application:** GlaceEMR
**Module:** CDA / QRDA / XDR / ADT Inbound
**Backend:** Spring Boot + Legacy JSP/Java

---

# Issue Summary

Errors were observed during ADT inbound processing for Glenwood.

Investigation focused on CDA/XDR validation utilities and identified repeated usage of the following OID across multiple modules:

```text id="mn7c0m"
2.16.840.1.113883.4.6
```

This OID corresponds to:

* NPI (National Provider Identifier)
* Assigned Entity Identifier
* Represented Organization Code System
* Provider Identification references in CDA/CCD/XDR generation

The same hardcoded OID was found in:

* Spring Boot backend
* Legacy JSP/Java modules
* QRDA export services
* CDA validator services
* XDR dummy request generators
* CCD utilities

---

# Root Cause Analysis

The ADT inbound validation failures were traced to inconsistent or duplicated usage of provider/entity OIDs across:

* CDA generation
* QRDA export
* XDR request formation
* Assigned author/entity construction
* Represented organization metadata

The common OID repeatedly used:

```text id="h2zmk1"
2.16.840.1.113883.4.6
```

represents:

```text id="12kkp5"
National Provider Identifier (NPI)
```

Improper reuse or malformed usage in certain payload structures likely caused inbound validation errors.

---

# Spring Boot – Files Investigated

---

# 1. CDA Validator – Entry Template IDs

## File

```text id="36j4ca"
/home/software/git/glaceemr_backend_new/src/main/java/com/glenwood/glaceemr/server/application/services/cdaValidator/CDAEntryTemplateIds.java
```

## Key Constants

```java id="mhm0iv"
public static final String NationalProviderIdentifier = "2.16.840.1.113883.4.6";
public static final String AssignedEntityId           = "2.16.840.1.113883.4.6";
```

Observation:

* Same OID used for both:

  * NPI
  * Assigned Entity ID

---

# 2. CDA Validator – CodeSystem

## File

```text id="r26y6r"
/home/software/git/glaceemr_backend_new/src/main/java/com/glenwood/glaceemr/server/application/services/cdaValidator/CodeSystem.java
```

## Key Constant

```java id="l7i9d4"
public static final String NPICodes = "2.16.840.1.113883.4.6";
```

## Utility Method

```java id="jwsc8v"
public static String getCodeSystemName(String code)
```

Observation:

* Reflection-based lookup used for code system resolution.
* Empty string returned if no match found.

Potential Risk:

* Missing/null mappings can silently pass validations.

---

# 3. DataConstantsUtilities

## File

```text id="b9yxzv"
/home/software/git/glaceemr_backend_new/src/main/java/com/glenwood/glaceemr/server/application/services/cdaValidator/DataConstantsUtilities.java
```

## Method

```java id="ayc12j"
public static String getRepresentOrgCodeSystem(){
    return "2.16.840.1.113883.4.6";
}
```

Observation:

* Represented Organization also uses NPI OID.

---

# 4. MIPS – CDABasicElementFactory

## File

```text id="3lh2l6"
/home/software/git/glaceemr_backend_new/src/main/java/com/glenwood/glaceemr/server/application/services/chart/MIPS/CDABasicElementFactory.java
```

## Assigned Entity Construction

```java id="ydn9m9"
test.setRoot("2.16.840.1.113883.4.6");
test.setExtension(providerNPI);
```

Observation:

* Provider NPI directly used as Assigned Entity root.

---

# 5. ExportQRDA – CDAEntryTemplateIds

## File

```text id="44s6aj"
/home/software/git/glaceemr_backend_new/src/main/java/com/glenwood/glaceemr/server/application/services/ExportQRDA/CDAEntryTemplateIds.java
```

## Constants

```java id="17jlwm"
public static final String NationalProviderIdentifier = "2.16.840.1.113883.4.6";
public static final String AssignedEntityId           = "2.16.840.1.113883.4.6";
```

Observation:

* Duplicate implementation across QRDA export modules.

---

# 6. ExportQRDA – DataConstantsUtilities

## File

```text id="p2z53t"
/home/software/git/glaceemr_backend_new/src/main/java/com/glenwood/glaceemr/server/application/services/ExportQRDA/DataConstantsUtilities.java
```

## Method

```java id="zh64d0"
public static String getRepresentOrgCodeSystem(){
    return "2.16.840.1.113883.4.6";
}
```

---

# 7. XDR Test Payload Generator

## File

```text id="n0lhix"
/home/software/git/glaceemr_backend_new/src/main/java/com/glenwood/glaceemr/XDR/TestXDR.java
```

## Problematic Patient ID Formatting

```java id="31qqm4"
extrinsicObject.getSlot().add(
    formSlot("sourcePatientId",
    "1^^^&2.16.840.1.113883.4.6&ISO")
);
```

## Additional Usage

```java id="4we4x3"
"1^^^&2.16.840.1.113883.4.6& ISO"
```

Observation:

* Inconsistent formatting:

  * `&ISO`
  * `& ISO`

Potential Cause:

* Invalid CX formatting in XDS/XDR metadata.
* Trailing spaces may fail validation.

---

# 8. IHE XDS_B Test Generator

## File

```text id="duj0z8"
/home/software/git/glaceemr_backend_new/src/main/java/ihe/iti/xds_b/_2007/Test.java
```

Same formatting issues identified:

```java id="5ah7n4"
"1^^^&2.16.840.1.113883.4.6& ISO"
```

and

```java id="mlj1n7"
"1^^^&2.16.840.1.113883.4.6&ISO"
```

---

# Legacy Application – Files Investigated

---

# 1. Legacy CDA DataConstantsUtilities

## File

```text id="q1k5xz"
/home/software/git/glacelegacy_master/WEB-INF/src/com/glenwood/glaceemr/cda/DataModels/DataConstantsUtilities.java
```

## Method

```java id="tr7b8g"
public static String getRepresentOrgCodeSystem(){
    return "2.16.840.1.113883.4.6";
}
```

---

# 2. CancerEventCarePlanSectionHandlerUtil

## File

```text id="w2cn3o"
/home/software/git/glacelegacy_master/WEB-INF/src/com/glenwood/glaceemr/cda/utils/CancerEventCarePlanSectionHandlerUtil.java
```

## Assigned Entity Usage

```java id="ny2lve"
assignedEntry.getId().add(
    this.formTemplateId(
        "2.16.840.1.113883.4.6",
        docorDetails.getOrgName().getNpi()
    )
);
```

## Null Flavor Case

```java id="5jz0h7"
ii.setRoot("2.16.840.1.113883.4.6");
ii.getNullFlavor().add("UNK");
```

Observation:

* Hardcoded NPI OID reused throughout assigned entity creation.

---

# 3. Legacy CDAEntryTemplateIds

## File

```text id="y0dr8h"
/home/software/git/glacelegacy_master/WEB-INF/src/com/glenwood/glaceemr/cda/utils/CDAEntryTemplateIds.java
```

## Constants

```java id="f4m7so"
public static final String NationalProviderIdentifier = "2.16.840.1.113883.4.6";
public static final String AssignedEntityId           = "2.16.840.1.113883.4.6";
```

---

# 4. Legacy CodeSystem

## File

```text id="sxd9az"
/home/software/git/glacelegacy_master/WEB-INF/src/com/glenwood/glaceemr/cda/utils/CodeSystem.java
```

## Constants

```java id="9m0uy7"
public static final String NPICodes = "2.16.840.1.113883.4.6";
```

---

# 5. GlaceCCDModule – Assigned Author/Entity

## File

```text id="wtm2sy"
/home/software/git/glacelegacy_master/WEB-INF/src/com/glenwood/hcare/GlaceCCDModule/CCDUtils/GlaceEntryElementsFactory.java
```

## Assigned Author

```java id="6m6f6q"
assignedAuthor.getId().add(
    this.formIIElement(
        "Glace-CCD-Author",
        "2.16.840.1.113883.4.6",
        null
    )
);
```

## Assigned Entity

```java id="mhmf3n"
entity.getId().add(
    this.formIIElement(
        "2.16.840.1.113883.4.6",
        null,
        "ProviderID"
    )
);
```

## Legal Authenticator

```java id="i7qv2w"
entity.getId().add(
    this.formIIElement(
        "2.16.840.1.113883.4.6",
        "99999999999",
        null
    )
);
```

Observation:

* Multiple generated entities use placeholder identifiers.
* Placeholder NPIs may fail strict ADT/XDR validation.

---

# Key Findings

| Area           | Observation                                     |
| -------------- | ----------------------------------------------- |
| CDA            | NPI OID hardcoded in multiple locations         |
| QRDA           | AssignedEntityId reused as NPI OID              |
| XDR            | Inconsistent `&ISO` formatting                  |
| Legacy         | Duplicate implementations across modules        |
| AssignedEntity | Placeholder IDs like `99999999999` used         |
| Validation     | Reflection lookup returns empty string silently |
| Patient IDs    | CX formatting inconsistencies detected          |

---

# Suspected Validation Failures

Possible inbound ADT/XDR validation failures include:

* Invalid CX patient identifier formatting
* Invalid assigning authority
* Improper use of NPI OID as organization identifier
* Missing assigning authority namespace
* Placeholder provider IDs
* Inconsistent whitespace in OIDs
* Empty code system resolution results

---

# Recommended Fixes

## 1. Centralize OID Constants

Create a single shared constants class:

```java id="3bqv8r"
public static final String NPI_OID = "2.16.840.1.113883.4.6";
```

Avoid duplicate definitions across:

* CDA Validator
* QRDA Export
* CCD Module
* Legacy utilities

---

# 2. Normalize CX Formatting

Use consistent format:

```text id="d1lxcb"
<ID>^^^&<OID>&ISO
```

Correct Example:

```text id="3u6t6f"
1^^^&2.16.840.1.113883.4.6&ISO
```

Avoid:

```text id="bgv5mu"
1^^^&2.16.840.1.113883.4.6& ISO
```

---

# 3. Remove Placeholder NPIs

Avoid:

```text id="7nhn9p"
99999999999
```

Use:

* Valid provider NPI
* Proper nullFlavor handling

---

# 4. Improve Validation Handling

Current implementation:

```java id="r4pqei"
return "";
```

Recommended:

* Throw validation exception
* Add warning/error logs
* Prevent silent failures

---

# 5. Add Shared Utility Layer

Suggested modules:

* OIDConstants
* CXFormatter
* ProviderIdentifierValidator
* CDAIdentifierUtil

---

# Conclusion

The ADT inbound errors appear related to inconsistent handling of:

* NPI OIDs
* Assigned Entity identifiers
* CX/XDS patient identifier formatting
* Placeholder provider identifiers

The issue exists across both:

* Spring Boot backend
* Legacy CDA/CCD generation modules

Primary corrective action should focus on:

1. Centralizing OID management
2. Standardizing identifier formatting
3. Eliminating placeholder IDs
4. Strengthening validation/error handling
5. Refactoring duplicate CDA utility implementations
