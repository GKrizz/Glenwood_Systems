
# MIPS PDF Setup & Verification Guide

## 📌 Purpose

This document provides step-by-step instructions to:

* Connect to SFTP server
* Locate MIPS/PDF JAR files
* Download required files
* Verify PDF generation components
* Understand overall flow

---

# 🔐 1. SFTP Connection

### Command to connect:

```bash
sftp -oPort=8444 glenwood@ftp.glaceemr.com
```

### Enter password when prompted:

```
glenwood@ftp.glaceemr.com's password:
```

### On successful connection:

```
Connected to ftp.glaceemr.com.
sftp>
```

---

# 📁 2. Navigate to Required Directory

### Check current directory:

```bash
pwd
```

### Move to Temp folder:

```bash
cd Temp
```

### Navigate to user folder:

```bash
cd Althaf
```

### Go to JARs folder:

```bash
cd Jars
```

---

# 📂 3. List Available JAR Files

```bash
ls *.jar
```

### Important JARs to look for:

* `MIPSJOBNew.jar`
* `MIPS.jar`
* `MIPS_2022.jar`
* `MIPS_status.jar`
* `PDFGenerator.jar`
* `GeneartePDF.jar`

---

# ⬇️ 4. Download Required JAR Files

### Download commands:

```bash
get MIPSJOBNew.jar
get MIPS.jar
get MIPS_2022.jar
get MIPS_status.jar
get PDFGenerator.jar
get GeneartePDF.jar
```

### Check local directory:

```bash
lpwd
```

---

# ⚠️ Note

* SFTP does NOT support commands like:

  ```
  jar tf
  grep
  ```
* Exit SFTP before running local commands.

---

# 🚪 5. Exit SFTP

```bash
exit
```

---

# 🔍 6. Inspect JAR Files (Local System)

### List contents of JAR:

```bash
jar tf PDFGenerator.jar
jar tf GeneartePDF.jar
```

---

# 🧩 7. Key Classes Identified

From analysis:

* `MIPS.class` → Core MIPS processing
* `HtmlTransformer.class` → HTML to PDF conversion
* `BIRTReportBean.class` → Report generation
* `ChannelSftp.class` → File transfer (SFTP)
* `MultipartUtility.class` → HTTP/File upload

---

# 🔄 8. MIPS PDF Flow

```text
Controller (sendPdf)
        ↓
Prepare data
        ↓
Call external JAR
        ↓
JAR Processing:
    - Fetch DB data
    - Generate HTML
    - Convert HTML → PDF
    - Upload via SFTP
        ↓
Return response/status
```

---

# 🔎 9. Code Verification (Project Side)

Search in project:

```bash
grep -r "sendPdf" .
grep -r "MIPS" .
grep -r "ProcessBuilder" .
grep -r "Runtime.getRuntime" .
```

---

# 🧠 10. Observations

* Multiple MIPS JAR versions exist
* PDF generation is handled externally (not in controller)
* Uses HTML → PDF conversion
* Likely uses BIRT reporting internally

---

# ❗ 11. Pending Checks

* Which JAR is actively used (likely `MIPSJOBNew.jar`)
* How JAR is triggered (API / cron / manual)
* Required input parameters
* Output location of generated PDF

---

# 📞 12. Next Steps

1. Review `sendPdf()` method in controller
2. Confirm execution flow with server admin (Vignesh)
3. Decompile JAR if deeper understanding is needed
4. Test JAR execution manually (if access available)

---
