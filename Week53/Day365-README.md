
# Day 365 – Errors on ADT Inbound for Glenwood

## Issue Summary: ADT Inbound Validation Error – Missing Organization ID Mapping

---

# Overview

CRISP Health reported inbound ADT processing failures for Glenwood-generated CDA/ADT messages.

The inbound system rejected certain messages because the following Organization ID OID was not mapped in their receiving system:

```text id="j3d0v4"
2.16.840.1.113883.4.6
```

This value was present inside the:

```text id="zjlwm6"
representedOrganization/id
```

section of the generated CDA document.

As a result:

* Incoming ADT messages failed validation
* Messages were not processed by the receiving system
* Errors appeared on CRISP/Connie inbound processing

---

# Original Support Request

## Reported By

| Name          | Organization |
| ------------- | ------------ |
| Cody Williams | CRISP Health |

---

## Original Concern

```text id="3d9hyx"
It looks to be coming from the
representedOrganizationID of
2.16.840.1.113883.4.6 not being mapped.

Is this a value we need to map moving forward?
```

---

# Problem Description

The generated CDA/ADT payload contains:

```xml id="4e5s1x"
<representedOrganization>
    <id root="2.16.840.1.113883.4.6"/>
</representedOrganization>
```

The receiving system expected a mapped Organization Identifier OID but did not recognize:

```text id="5qj9ho"
2.16.840.1.113883.4.6
```

This caused:

* Organization validation failures
* Mapping exceptions
* ADT message rejection

---

# Important Clarification

The OID:

```text id="bxjlwm"
2.16.840.1.113883.4.6
```

is officially assigned to:

```text id="u1p2fv"
National Provider Identifier (NPI)
```

It is intended for:

* Provider identifiers
* Individual clinician NPI references

It is **not typically used as an Organization Identifier**.

---

# Root Cause

The Glenwood CDA generation logic reused the NPI OID:

```text id="0jlwmx"
2.16.840.1.113883.4.6
```

for multiple unrelated purposes, including:

* AssignedEntity IDs
* RepresentedOrganization IDs
* Provider IDs
* Organization code systems

This caused downstream systems to interpret organization identifiers incorrectly.

---

# Example of Incorrect Usage

## Generated CDA Structure

```xml id="4xq8c9"
<representedOrganization>
    <id root="2.16.840.1.113883.4.6"/>
</representedOrganization>
```

Issue:

* Receiver interprets this as Organization OID
* But OID actually represents NPI namespace
* No organization mapping exists on receiver side

---

# Why Receiving System Failed

CRISP/Connie inbound validation relies on:

* Known Organization OID mappings
* Assigning authority validation
* Trusted identifier namespaces

Since:

```text id="dbn9f4"
2.16.840.1.113883.4.6
```

was not configured as an Organization OID in their system:

* Validation failed
* Message processing stopped
* ADT inbound queue generated errors

---

# CDA File Reference

Example generated file:

```text id="y2p8rm"
http://localhost:8080/GlaceMaster/shared/log/CDAFiles/cda_Outbox/OUT-05-19-2026-03-06-58-142-1.cda
```

---

# Technical Investigation

Earlier code analysis already identified widespread hardcoded reuse of:

```text id="lx2v9r"
2.16.840.1.113883.4.6
```

across:

* CDA Validator
* QRDA Export
* CCD Generation
* XDR Payloads
* AssignedEntity creation
* RepresentedOrganization generation

---

# Affected Areas

| Component              | Usage                    |
| ---------------------- | ------------------------ |
| CDAEntryTemplateIds    | AssignedEntityId         |
| DataConstantsUtilities | RepresentedOrgCodeSystem |
| CodeSystem.java        | NPICodes                 |
| CDABasicElementFactory | AssignedEntity root      |
| Legacy CCD modules     | Organization identifiers |
| XDR test payloads      | Patient/organization IDs |

---

# Main Technical Issue

The same OID was incorrectly reused for:

| Purpose                     | Correct? |
| --------------------------- | -------- |
| Provider NPI                | YES      |
| Organization Identifier     | NO       |
| Represented Organization ID | NO       |
| Facility Identifier         | NO       |

---

# Correct Approach

## Provider Identifiers

Use:

```text id="pjc9yt"
2.16.840.1.113883.4.6
```

ONLY for:

* NPI identifiers
* Individual providers

---

## Organization Identifiers

Represented organizations should instead use:

* Facility OID
* Organization assigning authority
* HIE-assigned organization namespace
* Valid enterprise OID

Example:

```xml id="ry6z1v"
<representedOrganization>
    <id root="1.2.840.x.x.x.x"/>
</representedOrganization>
```

---

# Receiver-Side Question

## “Do we need to map this value moving forward?”

### Answer

Technically they *can* map it temporarily, but this is not the ideal long-term solution.

Reason:

```text id="s0f5xp"
2.16.840.1.113883.4.6
is an NPI namespace,
not an organization namespace.
```

The better fix should happen on the sender side (Glenwood CDA generation).

---

# Recommended Fixes

# 1. Sender-Side Fix (Preferred)

Update CDA generation logic so:

```text id="0ll8tp"
representedOrganization/id
```

uses a proper organization/facility OID instead of the NPI OID.

---

# 2. Separate OID Usage

Avoid reusing:

```text id="vjlwm9"
2.16.840.1.113883.4.6
```

for:

* Organization IDs
* Facility IDs
* Enterprise identifiers

---

# 3. Centralize Identifier Configuration

Create dedicated constants:

```java id="91qvte"
public static final String NPI_OID
public static final String ORGANIZATION_OID
public static final String FACILITY_OID
```

instead of hardcoded reuse.

---

# 4. Add Validation

Before generating outbound CDA:

* Validate identifier type
* Validate assigning authority
* Validate organization namespace

Prevent invalid CDA generation.

---

# 5. Receiver-Side Temporary Workaround

CRISP/Connie may temporarily:

* Map `2.16.840.1.113883.4.6`
* Allow inbound processing

But this should be treated only as:

```text id="5jjlwm"
temporary compatibility support
```

not the final architectural solution.

---

# Impact

| Area                    | Impact   |
| ----------------------- | -------- |
| ADT Inbound             | Failed   |
| CDA Validation          | Failed   |
| Message Processing      | Blocked  |
| HIE Integration         | Affected |
| Organization Mapping    | Missing  |
| CRISP/Connie Processing | Rejected |

---

# Final Conclusion

The inbound ADT failures were caused by incorrect use of the NPI OID:

```text id="5zpjc6"
2.16.840.1.113883.4.6
```

inside:

```text id="v1h8od"
representedOrganization/id
```

The receiving HIE system expected a valid Organization OID mapping but instead received an NPI namespace OID.

This resulted in:

* Validation failures
* Organization mapping errors
* Rejected inbound ADT messages

---

# Resolution Direction

## Recommended Long-Term Solution

✔ Correct sender-side CDA generation
✔ Use proper Organization/Facility OIDs
✔ Stop reusing NPI namespace for organization identifiers
✔ Centralize OID management across CDA modules

---
