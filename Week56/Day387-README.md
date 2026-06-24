# CareQuality TIFF Viewer Support Enhancement

## Overview

This document describes the investigation, implementation, testing, and validation of TIFF document viewing support in the CareQuality document viewer.

The enhancement was implemented to allow users to preview TIFF/TIFF documents directly in the UI while preserving the ability to download the original TIFF file.

The work involved backend TIFF-to-PNG conversion, frontend viewer enhancements, download preservation, testing strategies, and future considerations for multi-page TIFF support.

---

## Background / Context

### Existing Behavior

CareQuality documents are retrieved and stored in:

```text
carequality_patient_document_details
```

The document viewer (`CDAFileView.jsp`) already supported:

* CDA
* PDF
* JPG
* PNG

For TIFF files:

```javascript
else if(fileExtension=="tif"){
    document.getElementById("imageContainer").style.display = "flex";
    document.getElementById("imageContainer").innerHTML =
        "<tr><td><b>click here to download</b></td></tr>"
        + "<tr><td><b><a href='"+fileRootPath+""+CDAFilename+"'>"
        + filename
        + "</a></b></td></tr>";
}
```

Users could only download TIFF files.

No preview capability existed.

---

## Problem Statement

### Issue

CareQuality documents received as TIFF files could not be viewed inside the application.

Users were forced to:

1. Download TIFF file
2. Open externally
3. View content

### Business Impact

* Poor user experience
* Inconsistent behavior compared to PDF/JPG/PNG documents
* Additional workflow steps for users reviewing CareQuality documents

---

## Impact

### Before Fix

| File Type | Supported     |
| --------- | ------------- |
| CDA       | Yes           |
| PDF       | Yes           |
| JPG       | Yes           |
| PNG       | Yes           |
| TIFF      | Download Only |

### After Fix

| File Type | Supported       |
| --------- | --------------- |
| CDA       | Yes             |
| PDF       | Yes             |
| JPG       | Yes             |
| PNG       | Yes             |
| TIFF      | View + Download |

---

## Environment Details

### Application

```text
Glace EMR
CareQuality Integration
```

### Backend

```text
Java
JSP
Servlets
PostgreSQL
```

### Shared Folder

```text
/mnt/vs22bshared/d2desktop
```

### Relevant Paths

```text
/mnt/vs22bshared/d2desktop/Attachments
```

### Source Files

#### Backend

```text
/home/software/git/glacelegacy_master/WEB-INF/src/com/glenwood/hcare/actions/chart/filemanager/CareQualityCommunication.java
```

#### Viewer

```text
/home/software/git/glacelegacy_master/jsp/CDAFileViewer/CDAFileView.jsp
```

#### CareQuality UI

```text
/home/software/git/glacelegacy_master/jsp/XCAQuery.jsp
```

---

## System Components Involved

### Backend Components

#### CareQualityCommunication.java

Responsible for:

* Document retrieval
* File download
* File storage
* Viewer response preparation

Methods involved:

```java
retrieveDocs()
downloadData()
convertTiffToPng()
```

---

### Frontend Components

#### XCAQuery.jsp

Responsible for:

* Retrieving document metadata
* Launching viewer

Functions:

```javascript
getDocument()
showDocument()
```

---

### Viewer

#### CDAFileView.jsp

Responsible for:

* Displaying CDA
* Displaying PDF
* Displaying Images
* TIFF preview rendering

---

## Investigation Timeline

### Phase 1 – Initial Issue

Users reported:

```text
TIFF documents cannot be viewed.
Only download link appears.
```

Existing logic:

```javascript
else if(fileExtension=="tif")
```

showed:

```text
click here to download
```

only.

---

### Phase 2 – Viewer Enhancement Design

Decision:

```text
Convert TIFF → PNG
Display PNG
Keep original TIFF downloadable
```

Reason:

Browsers have poor TIFF support.

PNG provides:

* Native rendering
* Better compatibility
* Simpler implementation

---

### Phase 3 – Backend Conversion

Added TIFF conversion logic.

New method:

```java
private void convertTiffToPng(String tifPath, String pngPath)
```

Implementation:

```java
BufferedImage bufferedImage = ImageIO.read(file);

ImageIO.write(bufferedImage, "png", pngFile);
```

---

### Phase 4 – Existing TIFF Support

Discovered:

Some TIFFs already existed in shared folder.

Need:

```text
Convert existing TIFF files when viewed.
```

Added conversion inside:

```java
retrieveDocs()
```

Logic:

```java
if(filePath.endsWith(".tif") || filePath.endsWith(".tiff"))
```

Convert TIFF if PNG does not exist.

---

### Phase 5 – Download Preservation

Problem:

Viewer now uses PNG path.

Without additional changes:

```text
Download button would download PNG.
```

Requirement:

```text
View PNG
Download TIFF
```

Solution:

Added:

```java
String downloadPath = filePath;
```

Before changing filePath to PNG.

Response:

```java
responseString.put("filePath", filePath);
responseString.put("downloadPath", downloadPath);
```

---

### Phase 6 – UI Enhancement

Viewer updated.

Previous:

```text
Download only
```

New:

```text
PNG preview
Download button
```

Implementation:

```javascript
<a href='"+fileRootPath+""+downloadPath+"'
download='"+filename.replace('.png','.tif')+"'>
Download
</a>
```

---

### Phase 7 – Validation

Senior review:

```text
Looks fine da.
Give download button.
If multiple tifs come.
Show me that also once.
```

Result:

Current implementation approved pending multi-page TIFF validation.

---

## Root Cause Analysis

### Root Cause

Application had no TIFF rendering capability.

Viewer logic explicitly treated TIFF files as:

```text
Download-only files.
```

No conversion mechanism existed.

No browser-renderable format generated.

---

### Contributing Factors

* Browser TIFF support limitations
* Viewer implementation only supported:

  * JPG
  * PNG
  * PDF
  * CDA

---

## Detailed Technical Findings

### Existing TIFF Handling

```javascript
else if(fileExtension=="tif")
```

Result:

```text
Download only
```

---

### New TIFF Handling

Flow:

```text
TIFF
 ↓
Backend Conversion
 ↓
PNG
 ↓
Viewer
 ↓
Original TIFF Download
```

---

## Code Changes

---

### File

```text
CareQualityCommunication.java
```

---

### Change #1

Added:

```java
convertTiffToPng()
```

```java
private void convertTiffToPng(String tifPath,String pngPath)
```

Purpose:

```text
Convert TIFF → PNG
```

---

### Change #2

Modified:

```java
downloadData()
```

Before:

```text
Stored TIFF
```

After:

```text
Store TIFF
Generate PNG
```

Logic:

```java
if(contenttype.contains("tiff"))
```

---

### Change #3

Modified:

```java
retrieveDocs()
```

Before:

```text
Existing TIFF remained TIFF
```

After:

```text
Existing TIFF converted to PNG on demand
```

Logic:

```java
if(!pngFile.exists())
{
    convertTiffToPng(...)
}
```

---

### Change #4

Added:

```java
downloadPath
```

Purpose:

```text
Preserve TIFF download path.
```

---

### Change #5

Modified:

```text
CDAFileView.jsp
```

New viewer:

```javascript
if(CDAFilename.toLowerCase().includes(".png"))
```

Features:

* PNG display
* Download button
* Scrollable viewer

---

## SQL Analysis and Scripts

### Investigation Queries

#### Find TIFF Files

```bash
find /mnt/vs22bshared/d2desktop/Attachments \
-type f \( -iname "*.tif" -o -iname "*.tiff" \)
```

Purpose:

```text
Locate TIFF documents.
```

---

#### Find CQ Records

```sql
select *
from carequality_patient_document_details
where carequality_patient_document_details_patient_id=1513;
```

Purpose:

```text
Reference existing CareQuality records.
```

---

#### Find TIFF References

```sql
select
carequality_patient_document_details_patient_id,
carequality_patient_document_details_document_id,
carequality_patient_document_details_file_path
from carequality_patient_document_details
where lower(carequality_patient_document_details_file_path) like '%.tif%'
   or lower(carequality_patient_document_details_file_path) like '%.tiff%';
```

Purpose:

```text
Identify TIFF-backed documents.
```

---

### Test Data Insert

```sql
INSERT INTO carequality_patient_document_details
(
    carequality_patient_document_details_patient_id,
    carequality_patient_document_details_service_start_time,
    carequality_patient_document_details_service_stop_time,
    carequality_patient_document_details_creation_time,
    carequality_patient_document_details_evaluation,
    carequality_patient_document_details_normal,
    carequality_patient_document_details_document_id,
    carequality_patient_document_details_repository_id,
    carequality_patient_document_details_community_id,
    carequality_patient_document_details_gateway,
    carequality_patient_document_details_author_name,
    carequality_patient_document_details_file_name,
    carequality_patient_document_details_file_path,
    carequality_patient_document_details_file_extension,
    is_document_downloaded
)
VALUES
(
    3237,
    now(),
    now(),
    now(),
    '',
    '',
    '1.2.840.114350.1.13.137.2.7.8.688883.186770508',
    '2.16.840.1.113883.4.391.1000',
    '2.16.840.1.113883.4.391.1000.30209',
    'surescripts',
    'Methodist Hospitals',
    'Hospital Encounter',
    '/Attachments/3237/1_2_840_114350_1_13_137_2_7_8_688883_186770508.tif',
    'tiff',
    1
);
```

Purpose:

```text
Testing TIFF viewer functionality.
```

Type:

```text
Validation Query
```

---

## Validation and Testing

### Functional Validation

Verified:

✅ TIFF file present

```text
/Attachments/3237/
```

✅ TIFF conversion executed

Expected:

```text
TIFF
PNG
```

Both files exist.

---

### UI Validation

Verified:

* PNG preview displayed
* Download button visible
* Download returns TIFF

---

### Download Validation

Verified:

```text
Viewer = PNG
Download = TIFF
```

---

## Risks and Side Effects

### Current Limitation

Current implementation:

```java
BufferedImage bufferedImage = ImageIO.read(file);
```

Potential issue:

```text
Multi-page TIFF may only render first page.
```

---

### Browser Compatibility

Low risk.

PNG rendering supported across browsers.

---

### Storage Impact

Every TIFF may generate:

```text
TIFF
PNG
```

Result:

```text
Additional disk usage.
```

---

## Pending Work

### Multi-page TIFF Validation

Requested by reviewer.

Need to verify:

```text
Single-page TIFF
Multi-page TIFF
```

Questions:

* Does ImageIO load all pages?
* Does viewer show all pages?

Current expectation:

```text
Only first page rendered.
```

Needs validation.

---

### Potential Enhancement

Future improvement:

```text
Convert each TIFF page to individual PNG.
```

Or:

```text
Generate PDF preview.
```

---

## Lessons Learned

### Technical

1. Browser TIFF support is unreliable.
2. PNG conversion is a practical viewer solution.
3. Preserve original document format for download.
4. Existing documents require conversion-on-demand logic.

---

### Debugging

1. Check existing files before implementing new storage logic.

2. Validate both:

   * newly downloaded files
   * existing files

3. Maintain separate:

```text
Viewer Path
Download Path
```

to avoid regressions.

---

### Future Recommendations

### Logging

Add logs:

```java
TIFF detected
PNG generated
PNG exists
Download path used
```

---

### Monitoring

Monitor:

```text
PNG generation failures
Large TIFF processing times
Disk growth due to PNG files
```

---

## Final Outcome

### Implemented

✅ TIFF viewer support

✅ Automatic TIFF → PNG conversion

✅ Existing TIFF conversion support

✅ Original TIFF download support

✅ UI preview enhancement

---

### Remaining Validation

⚠ Multi-page TIFF behavior still needs testing and confirmation before final production sign-off.
