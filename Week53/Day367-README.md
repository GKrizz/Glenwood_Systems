
# Day 365 – CareQuality / XCA Document Query & Retrieval Flow

# Overview

This document explains the complete CareQuality / XCA integration flow used in GlaceEMR for:

* Querying external patient documents
* Retrieving CDA/PDF documents
* Viewing downloaded records
* Database tracking
* Legacy ↔ Spring ↔ CareQuality gateway interaction

The integration uses:

* Legacy Struts application (`glacelegacy_master`)
* Spring Boot backend (`glaceemr_backend_new`)
* CareQuality / Surescripts HIE Gateway
* IHE XCPD + XCA standards

---

# Main Entry Point

## Care Quality UI

When user clicks:

```text
Care Quality
```

Browser calls:

```text
https://ccw.glaceemr.com/Glace/jsp/chart/patientdetails/CareQualityCommunication.Action
```

Request:

```json
{
  "mode": "1",
  "fetchDocuments": "false",
  "patientId": "104",
  "GlaceAjaxRequest": "true"
}
```

Response:

```json
{
  "status":"2"
}
```

---

# Action Mapping

File:

```text
/home/software/git/glacelegacy_master/WEB-INF/conf/ActionMappings.properties
```

Mapping:

```properties
# CareQuality Communication
CareQualityCommunication.Action=com.glenwood.hcare.actions.chart.filemanager.CareQualityCommunication

XCAQuery.Screen=/jsp/XCAQuery.jsp
XCADoc.Screen=/jsp/XCADoc.jsp
PatientNotMappedError.Screen=/jsp/cda/incorporation/PatientNotMappedError.jsp
```

---

# Core Flow Architecture

```text
Browser
   │
   ▼
Legacy Struts App
(CareQualityCommunication.java)
   │
   ▼
Spring Boot Backend
(CareQualityController.java)
   │
   ▼
CareQuality Gateway
(HIE Discovery APIs)
   │
   ▼
Surescripts / CareQuality Network
   │
   ├── XCPD
   ├── ITI-38 (Document Query)
   └── ITI-39 (Document Retrieve)
```

---

# Main Modes

| Mode | Purpose                   |
| ---- | ------------------------- |
| 1    | Initialize CareQuality UI |
| 2    | Query patient documents   |
| 3    | Retrieve/View document    |
| 6    | XCPD status check         |
| 7    | Polling status            |

---

# MODE 2 – Query Patient Documents

# Browser Request

```text
POST CareQualityCommunication.Action?mode=2&patientId=104
```

Request:

```json
{
  "mode": "2",
  "patientId": "104",
  "GlaceAjaxRequest": "true"
}
```

---

# Response Example

```json
{
  "lastUpdatedDate": "05/20/2026 03:58:59 am",
  "data": [
    {
      "download": "0",
      "filename": "Summarization Of Episode",
      "repositoryid": "2.16.840.1.113883.3.9168",
      "authorname": "CRISP",
      "interval": "0",
      "documentid": "6d46eab1-6a47-7c4c-476a-4c7cb1ea466d",
      "communityid": "urn:oid:2.16.840.1.113883.3.9168",
      "gateway": "surescripts"
    }
  ]
}
```

---

# Legacy Flow – queryPatient()

File:

```text
CareQualityCommunication.java
```

Method:

```java
queryPatient()
```

Flow:

```text
Browser
   │
   └─► POST CareQualityCommunication.Action?mode=2
               │
               ▼
       queryPatient()
```

---

# queryPatient() Logic

## Step 1 – Check Query Status

Table:

```sql
cq_query_log
```

Checks:

* already queried?
* status completed?
* stale data?
* polling required?

---

## Step 2 – Check Existing Documents

Table:

```sql
carequality_patient_document_details
```

Query:

```sql
SELECT *
FROM carequality_patient_document_details
WHERE patient_id=102
ORDER BY service_start_time DESC;
```

---

# Decision Tree

```text
Fresh data exists?
   YES → return DB data directly

No data / stale / force refresh?
   YES → call getpatientDocs()
```

---

# getpatientDocs() Flow

```text
getpatientDocs()
   │
   ├── INSERT cq_query_log(status=1)
   │
   ├── POST Spring API
   │
   ├── Receive JSONArray of documents
   │
   ├── Store metadata in DB
   │
   └── UPDATE cq_query_log(status=2)
```

---

# Spring Backend Call

Legacy calls:

```text
{springURL}/api/emr/glacemonitor/CareQuality/QueryPatient
```

Example:

```text
POST /api/emr/glacemonitor/CareQuality/QueryPatient
```

---

# Spring Controller

File:

```text
CareQualityController.java
```

Calls:

```java
careQualityService.queryPatient()
```

Implementation:

```text
CareQualityServiceImpl.java
```

Method:

```java
queryPatient()
```

---

# Spring queryPatient() Internal Flow

## Step 1 – Fetch Patient Demographics

Repository:

```java
PatientRegistrationRepository
```

Fetches:

* firstName
* lastName
* DOB
* gender
* address
* SSN
* telecom

---

## Step 2 – Fetch Account Configuration

Calls SSO:

```text
http://sso.glaceemr.com/TestSSOAccess?accountId=xxx
```

Returns:

* acc_crm_id
* practiceName
* shared folder path

---

## Step 3 – Check cq_service Flag

Table:

```sql
cq_service
```

Determines:

| Flag   | Endpoint               |
| ------ | ---------------------- |
| 2      | locateFromGlaceForGlen |
| others | locateFromGlace        |

---

# External CareQuality Gateway Call

Gateway:

```text
https://carequality.glaceemr.com:444/HIE/discovery/
```

Endpoints:

```text
locateFromGlace
locateFromGlaceForGlen
```

---

# Actual HIE Standards Used

| Standard | Purpose                      |
| -------- | ---------------------------- |
| XCPD     | Patient Discovery            |
| ITI-38   | Cross Gateway Document Query |
| ITI-39   | Cross Gateway Retrieve       |

---

# Returned Organizations

Examples:

* CRISP
* Hartford Healthcare
* Trinity Health
* UConn Health
* Epic Medical
* Surescripts

---

# Document Metadata Storage

Table:

```sql
carequality_patient_document_details
```

Stores:

* document_id
* repository_id
* community_id
* gateway
* author_name
* filename
* download flag
* filepath
* timestamps

---

# MODE 3 – Retrieve/View Document

# Browser View Click

Example:

```text
https://ccw.glaceemr.com/Glace/CareQualityCommunication.Action
?fromCareQuality=1
&mode=3
&gateWay=surescripts
&documentId=a7a20e8c...@@repo@@community
&patientId=102
```

---

# Response

```json
{
  "mode": 3,
  "extension": "cda",
  "filename": "a7a20e8c-2347-b781-4723-81b78c0ea2a7.cda",
  "filePath": "/Attachments/102/a7a20e8c-2347-b781-4723-81b78c0ea2a7.cda",
  "documentId": "a7a20e8c-2347-b781-4723-81b78c0ea2a7",
  "gateWay": "surescripts"
}
```

---

# retrieveDocs() Flow

File:

```text
CareQualityCommunication.java
```

Method:

```java
retrieveDocs()
```

---

# DocumentId Parsing

Example:

```text
a7a20e8c...@@2.16.840...@@2.16.840...
```

Split into:

| Part | Meaning         |
| ---- | --------------- |
| [0]  | document UUID   |
| [1]  | repositoryId    |
| [2]  | homeCommunityId |

---

# Download Check

Query:

```sql
SELECT
   is_document_downloaded,
   file_path
FROM carequality_patient_document_details
WHERE document_id=?
```

---

# Decision Logic

## Already Downloaded

```text
download = 1
```

Use existing file:

```text
/Attachments/102/<uuid>.cda
```

No external call made.

---

## Not Downloaded

```text
download = 0
```

Calls:

```java
downloadData()
```

---

# downloadData() Flow

Constructs URL:

```java
/api/emr/glacemonitor/CareQuality/retrieveDocument
```

Example:

```text
https://dev-springs.glaceemr.com/
glaceemr_backend_stable_v2/
api/emr/glacemonitor/CareQuality/retrieveDocument
```

---

# Spring retrieveDocument()

Controller:

```java
CareQualityController.java
```

Method:

```java
retrieveDocument()
```

Implementation:

```java
CareQualityServiceImpl.retrieveDocument()
```

---

# External Gateway Retrieve

Calls:

```text
https://carequality.glaceemr.com:444/HIE/discovery/retrieveDocument
```

Uses:

```text
IHE XCA ITI-39
```

---

# Returned Content

Gateway returns:

```text
Base64 encoded CDA/PDF/TIFF/JPEG
```

---

# File Type Detection

Logic:

| Content Type | Saved As |
| ------------ | -------- |
| TIFF         | .tif     |
| PDF          | .pdf     |
| JPEG         | .jpg     |
| XML/CDA      | .cda     |

---

# Example – PDF Document

Request:

```text
documentId=
1.2.840.114350...
```

Response:

```json
{
  "extension": "pdf",
  "filename": "1.2.840....pdf",
  "filePath": "/Attachments/104/1_2_840_....pdf"
}
```

DB Check:

```sql
SELECT
   is_document_downloaded,
   carequality_patient_document_details_file_path
FROM carequality_patient_document_details
WHERE carequality_patient_document_details_document_id=
'1.2.840.114350...';
```

Result:

```text
download = 1
filepath = /Attachments/104/xxx.pdf
```

---

# File Storage Location

Files saved under:

```text
{sharedFolderPath}/Attachments/<patientId>/
```

Examples:

```text
/Attachments/102/a7a20e8c....cda
/Attachments/104/1_2_840_....pdf
```

---

# CDA Viewer Flow

Browser loads:

```text
CDAXMLVewer.jsp
```

Parameters:

```json
{
  "xmlfiletype": "1",
  "CDAFilename": "/Attachments/102/xxx.cda"
}
```

---

# CDA Rendering

Components:

| Component          | Purpose             |
| ------------------ | ------------------- |
| CDAParserService   | Parse CDA XML       |
| CDATransformerUtil | XSLT transformation |
| CDAXMLVewer.jsp    | Render CDA HTML     |

---

# Rendered Output

User sees:

* Patient demographics
* Vital signs
* Labs
* Social history
* Results
* Processed format
* Raw/simple CDA view

---

# Main Database Tables

# cq_query_log

Tracks query execution status.

| Status | Meaning     |
| ------ | ----------- |
| 1      | In Progress |
| 2      | Completed   |
| 3      | Error       |

---

# carequality_patient_document_details

Stores:

* metadata
* repository/community IDs
* filepath
* download status

---

# Key APIs

## Query Documents

```text
/api/emr/glacemonitor/CareQuality/QueryPatient
```

Purpose:

```text
ITI-38 XCA Query
```

---

## Retrieve Document

```text
/api/emr/glacemonitor/CareQuality/retrieveDocument
```

Purpose:

```text
ITI-39 XCA Retrieve
```

---

# Full End-to-End Architecture

```text
Browser
   │
   ▼
Legacy Struts
(CareQualityCommunication.java)
   │
   ▼
Spring Boot
(CareQualityController.java)
   │
   ▼
CareQualityServiceImpl.java
   │
   ▼
carequality.glaceemr.com Gateway
   │
   ▼
Surescripts CareQuality Network
   │
   ├── XCPD
   ├── ITI-38
   └── ITI-39
   │
   ▼
CRISP / Epic / Trinity / UConn / Hartford
```

---

# Final Summary

The CareQuality integration works as a hybrid architecture:

## Legacy Layer

Responsible for:

* UI rendering
* user interaction
* document caching
* DB persistence
* local file handling

---

## Spring Backend

Responsible for:

* orchestration
* patient query processing
* external gateway communication
* retrieval APIs

---

## CareQuality Gateway

Responsible for:

* HIE communication
* XCPD patient discovery
* XCA query/retrieve
* Surescripts network interaction

---
