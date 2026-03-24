# 📄 Day 288 – MIPS Performance Job Timeout

## 🧩 Issue Summary

The **MIPSPerformanceJob** is getting a timeout due to a **502 Proxy Error** while calling the `getPatientsSeen` API.

---

## 🔗 Job URL

```
https://emrbatch.glaceemr.com/GlaceBatch/jobs/MIPSPerformanceJob
```

### Parameters

```
mode=3
reportingYear=2025
quartzId=weokhgsmnb_mwcw
accid=mwcw
isMonthlyReport=false
```

---

## 🌐 Backend Service URL

```
http://172.18.24.162/glaceemr_backend_epcs
```

---

## ⚠️ Error Details

### Log Snippet

```
2026-03-02 01:46:04 WARN  RestTemplate:549 - GET request for 
"https://glace-gwt.glaceemr.com/glaceemr_backend_epcs/api/emr/glacemonitor/mipsperformance/getPatientsSeen
?dbname=mwcw&accountID=mwcw&mode=3&reportingYear=2025" 

resulted in 502 (Proxy Error)
```

### Exception

```
org.springframework.web.client.HttpServerErrorException: 502 Proxy Error
```

---

## 🔍 Observations

* Job starts successfully and begins step: `loadPatients`
* Failure occurs during API call:

  ```
  /getPatientsSeen
  ```
* External (proxy) URL fails:

  ```
  https://glace-gwt.glaceemr.com
  ```
* Internal service works via direct IP:

  ```
  http://172.18.24.162
  ```

---

## 🧪 API Testing (cURL)

### 1. Get Patients Seen

```
curl -X GET "http://172.18.24.162/glaceemr_backend_epcs/api/emr/glacemonitor/mipsperformance/getPatientsSeen?dbname=mwcw&accountID=mwcw&mode=3&reportingYear=2025"
```

---

### 2. Generate & Validate QDM

```
curl -X GET "http://172.18.24.162/glaceemr_backend_epcs/api/emr/glacemonitor/mipsperformance/generateAndValidateQDM?dbname=mwcw&accountId=mwcw&patientID=12846&providerId=1&reportingYear=2025"
```

---

### 3. Calculate MIPS Performance

```
curl -X GET "http://172.18.24.162/glaceemr_backend_epcs/api/emr/glacemonitor/mipsperformance/calculateMIPSPerformance?reportingYear=2025&accountID=mwcw&isMonthlyReport=false&dbname=mwcw"
```

---

## 🧠 Root Cause (Likely)

* Proxy (`glace-gwt`) returning **502 Bad Gateway**
* Possible reasons:

  * Backend service timeout
  * Proxy routing issue
  * Service not reachable via domain
  * Load balancer / gateway misconfiguration

---
