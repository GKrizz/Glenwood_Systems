
# CareQuality TIFF Viewer Enhancement - Multi-Page TIFF Support Investigation

## Overview

This document describes the investigation, implementation, testing, and debugging performed to enhance the CareQuality document viewer to support TIFF documents.

Initially, the application only supported displaying PDF, CDA, JPG, and PNG documents. TIFF files required users to download them manually before viewing, which was not user-friendly.

The enhancement aimed to:

* Allow TIFF files to be viewed directly inside the CareQuality UI.
* Preserve the ability to download the original TIFF file.
* Extend support from single-page TIFFs to multi-page TIFFs.
* Investigate performance issues encountered while converting large multi-page TIFFs into PNG images.

---

# Background / Context

## Business Request

Support requested the following enhancement:

> TIFF files should be viewable directly in the application.
>
> Practices should not be forced to download TIFF files to view them.

Original Support Request

```
TIFF, we need to see if we can get a viewer.
I can't ask practice to download TIFFs to view.
```

## Existing Behavior

When a CareQuality document was downloaded:

* PDF → displayed correctly
* CDA → displayed correctly
* JPG → displayed correctly
* PNG → displayed correctly
* TIFF → user had to download manually

No browser rendering existed for TIFF.

---

# Problem Statement

The CareQuality viewer had no native TIFF rendering capability.

Although browsers cannot directly render TIFF images consistently, the application stored TIFF files locally after retrieval.

The solution selected was:

```
TIFF
        │
        ▼
Convert to PNG
        │
        ▼
Display PNG inside browser
        │
        ▼
Allow download of original TIFF
```

---

# Impact

Without this enhancement:

* Users had to download TIFF files
* Poor user experience
* Multiple clicks
* Not suitable for CareQuality workflow

Business expectation:

* Click "View"
* TIFF should open directly
* Download button should still download the original TIFF

---

# Environment Details

| Component    | Value                         |
| ------------ | ----------------------------- |
| Application  | Glace Legacy                  |
| Module       | CareQuality                   |
| Class        | CareQualityCommunication.java |
| Method       | retrieveDocs()                |
| Method Added | convertTiffToPng()            |
| Language     | Java                          |
| Libraries    | ImageIO                       |
| Storage      | Shared Folder (/Attachments)  |
| Frontend     | JSP                           |

---

# System Components Involved

## Java Class

```
CareQualityCommunication.java
```

Methods involved

```
retrieveDocs()

downloadData()

convertTiffToPng()
```

---

# Initial Implementation

When retrieveDocs() detects

```
.tif
```

or

```
.tiff
```

it now performs

```
TIFF

↓

Convert TIFF → PNG

↓

Return PNG path to UI

↓

Keep original TIFF path for Download button
```

---

# UI Changes

Original image rendering supported

```
JPG
PNG
```

Updated logic

```
if(fileExtension=="jpg" || fileExtension=="png")
```

PNG generated from TIFF is displayed.

Download button downloads

```
original.tif
```

instead of

```
generated.png
```

---

# Initial TIFF Conversion Method

Originally

```java
private void convertTiffToPng(String tifPath, String pngPath) throws Exception {
    File file = new File(tifPath);

    BufferedImage bufferedImage = ImageIO.read(file);

    if(bufferedImage == null){
        throw new Exception("ImageIO.read() returned NULL");
    }

    ImageIO.write(bufferedImage,"png",new File(pngPath));
}
```

---

## Limitation

`ImageIO.read()` only reads the **first page** of a multi-page TIFF.

Result:

```
TIFF

Page 1
Page 2
Page 3
Page 4
Page 5
Page 6

↓

PNG

Only Page 1
```

---

# Investigation

## Multi-page TIFF Analysis

Using ImageMagick

```bash
identify 1_2_840_114350_1_13_137_2_7_8_688883_186770508.tif
```

Output

```
[0]
[1]
[2]
[3]
[4]
[5]
```

Confirmed

```
6 pages
```

---

Another validation

```bash
identify file.tif | wc -l
```

Result

```
6
```

---

# Revised Implementation

Senior recommendation

Instead of

```
ImageIO.read()
```

Use

```
ImageReader
```

Advantages

* Reads page count
* Reads page dimensions
* Reads page-by-page
* Memory efficient

---

Flow

```
Open TIFF

↓

ImageReader

↓

Read metadata

↓

Read every page

↓

Merge vertically

↓

Generate PNG

↓

Return PNG
```

---

# Revised Conversion Method

Key improvements

## Pass 1

Read dimensions only

```
reader.getWidth()

reader.getHeight()
```

instead of

```
reader.read()
```

---

## Pass 2

Read one page

Draw

Release memory

Repeat

```
Page1

↓

Flush

↓

Page2

↓

Flush

↓

Page3
```

instead of storing

```
BufferedImage[]

```

for every page.

---

## Memory Optimization

Previous implementation

```
BufferedImage[] pages
```

held every page simultaneously.

Updated implementation

```
BufferedImage page

↓

Draw

↓

flush()

↓

Read next
```

Only one page remains in memory.

---

# Debug Logging Added

Temporary logs added

```java
===== ENTERED convertTiffToPng() =====

TIFF Path

PNG Path

TIFF Page Count

Reading page

Page Height

PNG Written

EXIT
```

Purpose

* Verify method execution
* Verify page count
* Verify image writing

Comment added

```java
// Temporary debug logs added to verify multi-page TIFF conversion.
// These logs will be removed after verification.
```

---

# Debug Output

Observed

```
ENTERED

TIFF Page Count : 6

Reading page 1

Reading page 2

Reading page 3

Reading page 4

Reading page 5

Reading page 6
```

This confirms

* method invoked
* TIFF reader working
* all six pages detected
* all six pages read successfully

---

# Performance Observation

Immediately after page 6

JVM produced

```
GC Pause

5677MB

↓

654MB
```

No further logs

```
PNG Written

EXIT
```

appeared.

---

# Root Cause Analysis

## Confirmed

The issue is **not** in reading TIFF pages.

Evidence

```
Page Count = 6

Reading page 1

...

Reading page 6
```

completed successfully.

---

## Suspected Bottleneck

Processing likely stalls after reading pages during one of these operations:

1. Large merged `BufferedImage` allocation.
2. PNG encoding (`ImageIO.write()`).
3. Memory pressure leading to heavy garbage collection.

This was **not conclusively proven** during the conversation, but the debug output indicates the slowdown occurs after page reading and before successful PNG completion.

---

# SQL Analysis and Scripts

## Investigation Queries

### Search TIFF Records

```sql
SELECT
    carequality_patient_document_details_patient_id,
    carequality_patient_document_details_document_id,
    carequality_patient_document_details_gateway,
    carequality_patient_document_details_file_extension,
    carequality_patient_document_details_file_path
FROM carequality_patient_document_details
WHERE carequality_patient_document_details_file_path LIKE '%.tif%';
```

Purpose

Find all TIFF documents.

Type

Investigation Query

---

### Find Latest TIFF Records

```sql
SELECT
    carequality_patient_document_details_patient_id,
    carequality_patient_document_details_document_id,
    carequality_patient_document_details_file_name,
    carequality_patient_document_details_creation_time
FROM carequality_patient_document_details
WHERE lower(carequality_patient_document_details_file_path) LIKE '%.tif%'
ORDER BY carequality_patient_document_details_creation_time DESC;
```

Purpose

Identify recent TIFF documents for testing.

Type

Validation Query

---

### Join with Patient Table

```sql
SELECT DISTINCT
    p.patient_registration_id,
    p.patient_registration_accountno,
    p.patient_registration_last_name,
    p.patient_registration_first_name,
    c.carequality_patient_document_details_file_name,
    c.carequality_patient_document_details_file_path
FROM carequality_patient_document_details c
JOIN patient_registration p
ON p.patient_registration_id =
c.carequality_patient_document_details_patient_id
WHERE lower(c.carequality_patient_document_details_file_extension)
LIKE '%tiff%'
OR lower(c.carequality_patient_document_details_file_path)
LIKE '%.tif%';
```

Purpose

Locate real patients having TIFF attachments.

Type

Investigation Query

---

# Testing

## Verified

✔ TIFF conversion method invoked

✔ TIFF page count detected

✔ Six pages read

✔ Download button still downloads original TIFF

✔ PNG generated for single-page TIFFs

---

## Pending

* Complete rendering of merged multi-page PNG.
* Resolve performance issue causing prolonged processing after page reading.
* Confirm successful PNG generation and display for large multi-page TIFFs.

---

# Risks

Potential risks include:

* Large TIFFs may require significant memory during merge.
* PNG generation for high-resolution documents can be CPU intensive.
* Long-running conversions may block the request thread and impact user experience.

---

# Pending Work

* Determine whether the slowdown occurs during merged image allocation or `ImageIO.write()`.
* Add finer-grained timing logs around image creation and PNG writing.
* Evaluate streaming or per-page rendering if merged PNG generation remains impractical.
* Remove temporary debug `System.out.println()` statements after investigation.

---

# Lessons Learned

* `ImageIO.read()` is suitable only for single-image TIFFs and ignores additional pages.
* `ImageReader` provides access to all pages in a multi-page TIFF and allows more efficient processing.
* Validating page count independently (e.g., with `identify`) is useful before debugging application code.
* Temporary logging around critical stages (entry, page count, page read, image write) helps isolate the exact stage where processing slows or fails.
* Large image processing can expose memory and performance issues even when the input file size appears moderate, because the in-memory decoded image is much larger than the compressed TIFF file.
