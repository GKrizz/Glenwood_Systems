
# Day 361 – MIPS Monthly Report

## Overview

The MIPS Monthly Report process is an automated batch workflow used to:

* Fetch all MIPS-enabled accounts
* Trigger MIPS PDF generation
* Email provider-wise MIPS performance reports

The orchestration is handled by:

```text id="u91vyy"
MIPSPDF.jar
```

while the actual report generation and emailing logic is handled inside the Spring backend.

---

# API Endpoint

## Spring Backend API

```text id="bjlwm7"
[Glace_spring_URL or Glace_spring_public_URL]/api/emr/glacemonitor/mipsperformance/sendMIPSPDFviaEmail
```

---

# Sample API

```text id="ewzt0x"
http://spring-portal.glaceemr.com/glaceemr_backend_portal/api/emr/glacemonitor/mipsperformance/sendMIPSPDFviaEmail?reportingYear=2026&accountID=mim&dbname=mim
```

---

# Request Method

```text id="4e0o57"
POST
```

---

# Request Parameters

| Parameter     | Description         |
| ------------- | ------------------- |
| reportingYear | MIPS reporting year |
| accountID     | Client account      |
| dbname        | Database name       |

---

# Request Payload

```json id="uhg6wa"
{
  "excludeHospitalVisit": false,
  "excludeERVisit": false,
  "excludeNursingHomeVisit": false
}
```

---

# CURL Example

```bash id="1sg5pd"
curl -X POST \
"http://spring-portal.glaceemr.com/glaceemr_backend_portal/api/emr/glacemonitor/mipsperformance/sendMIPSPDFviaEmail?reportingYear=2026&accountID=mim&dbname=mim" \
-H "Content-Type: application/json" \
-d '{
  "excludeHospitalVisit": false,
  "excludeERVisit": false,
  "excludeNursingHomeVisit": false
}'
```

---

# High-Level Architecture

```text id="5e9x9o"
MIPSPDF.jar
    ↓
SSO APIs
    ↓
Identify Spring URL
    ↓
Trigger Spring API
    ↓
Generate PDF
    ↓
Email Providers
```

---

# Complete Flow

# 1. MIPSPDF.jar Starts

The batch job starts either:

* Manually
* Scheduler/Cron

Purpose:

* Trigger MIPS PDF generation for all enabled accounts.

---

# 2. Fetch MIPS Enabled Accounts

Jar calls SSO API:

```text id="7p2m0s"
https://sso.glaceemr.com/SearchAccounts?mips=1
```

Purpose:

* Retrieve all accounts configured for MIPS processing.

---

# 3. Resolve Spring Backend URL

For each account, jar calls:

```text id="xgn6qj"
https://sso.glaceemr.com/TestSSOAccess?accountId=<accountid>
```

Purpose:

* Identify the correct Spring backend URL for the account.

Example:

```text id="rqlg2k"
spring-portal.glaceemr.com
```

---

# 4. Build Dynamic API URL

Jar dynamically constructs:

```text id="c7s5c6"
[SpringURL]/api/emr/glacemonitor/mipsperformance/sendMIPSPDFviaEmail?reportingYear=2026&accountID=<accountid>&dbname=<accountid>
```

Example:

```text id="dxtjkh"
http://spring-portal.glaceemr.com/glaceemr_backend_portal/api/emr/glacemonitor/mipsperformance/sendMIPSPDFviaEmail?reportingYear=2026&accountID=mim&dbname=mim
```

---

# 5. Send POST Request

Payload:

```json id="c7o2y4"
{
  "excludeHospitalVisit": false,
  "excludeERVisit": false,
  "excludeNursingHomeVisit": false
}
```

Purpose:

* Control exclusion filters during MIPS calculation.

---

# 6. Spring Backend Processing

Spring backend performs:

## Provider Fetch

* Retrieves active providers
* Validates MIPS eligibility

---

## MIPS Calculation

Calculates:

* Quality
* PI
* IA
* Cost
* Final MIPS score

---

## Dashboard HTML Generation

Builds provider-wise dashboard HTML dynamically.

---

## PDF Generation

HTML converted to PDF using:

```text id="gkhg5z"
wkhtmltopdf
```

---

# 7. Email PDF to Providers

Generated PDF sent via:

* Mailer service
* Provider email configuration

Attachments include:

* Monthly MIPS dashboard PDF
* Provider performance details

---

# 8. Logging

Jar logs:

* Account processing
* API response
* Success/failure status
* Exceptions

Console output used for monitoring batch execution.

---

# Overall Responsibility Split

| Component      | Responsibility                  |
| -------------- | ------------------------------- |
| MIPSPDF.jar    | Batch orchestrator              |
| SSO APIs       | Account + Spring URL resolution |
| Spring Backend | MIPS calculation                |
| Spring Backend | Dashboard HTML generation       |
| Spring Backend | PDF generation                  |
| Spring Backend | Email delivery                  |
| wkhtmltopdf    | HTML → PDF conversion           |

---

# Key APIs Used

## Fetch Accounts

```text id="0umz40"
https://sso.glaceemr.com/SearchAccounts?mips=1
```

---

## Fetch Spring URL

```text id="t0f3wj"
https://sso.glaceemr.com/TestSSOAccess?accountId=<accountid>
```

---

## Generate & Email MIPS PDF

```text id="r58w3y"
[SpringURL]/api/emr/glacemonitor/mipsperformance/sendMIPSPDFviaEmail
```

---

# Request Payload Options

| Field                   | Purpose                     |
| ----------------------- | --------------------------- |
| excludeHospitalVisit    | Exclude hospital visits     |
| excludeERVisit          | Exclude ER visits           |
| excludeNursingHomeVisit | Exclude nursing home visits |

---

# Summary

The monthly MIPS reporting system is designed as:

## Batch Trigger Layer

Handled by:

```text id="06md2k"
MIPSPDF.jar
```

Responsibilities:

* Fetch accounts
* Identify Spring backend
* Trigger APIs

---

## Business Logic Layer

Handled by:

```text id="2o2tqo"
Spring Backend
```

Responsibilities:

* MIPS score computation
* Dashboard generation
* PDF conversion
* Email delivery

---

# Final Workflow

```text id="gq7n1m"
Scheduler/Manual Trigger
        ↓
MIPSPDF.jar
        ↓
Fetch MIPS Accounts
        ↓
Resolve Spring URL
        ↓
Trigger Spring API
        ↓
Generate MIPS Dashboard
        ↓
Convert HTML → PDF
        ↓
Email Providers
        ↓
Log Results
```
