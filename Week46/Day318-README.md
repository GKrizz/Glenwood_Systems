# 🔷 What is Carequality Module?

👉 **Carequality** is a **health data exchange system** (used in the US healthcare ecosystem).

👉 In your application (Glace EMR), the **Carequality module** is used to:

➡️ **Connect with external hospitals/clinics**
➡️ **Fetch patient medical records from other organizations**
➡️ **Share patient data securely across systems**

📌 Think of it like:

> **“Google for patient records across hospitals”**

---

# 🎯 Main Purpose (Why we use this?)

### 1. 🏥 **Get Complete Patient History**

* Patient may visit multiple hospitals
* Your system can **pull records from other organizations**
* Example:

  * Lab results
  * Clinical notes
  * Reports

---

### 2. 🔄 **Interoperability (System-to-System Communication)**

* Different hospitals use different software
* Carequality ensures:
  ✔ Data flows between them
  ✔ Standard format (FHIR, HL7)

---

### 3. 👨‍⚕️ **Better Clinical Decisions**

* Doctor sees full history → makes better decisions
* Avoids duplicate tests

---

### 4. 📊 **Improves Healthcare Efficiency**

* No manual document sharing
* No repeated data entry
* Faster workflow

---

### 5. 🔐 **Secure Data Exchange**

* Uses:

  * Certificates
  * SAML
  * Secure APIs
* Ensures **HIPAA compliance**

---

# ⚙️ What Happens in Your System (Workflow)

## 🧭 Step-by-Step Flow

### 1. User clicks CQ icon in patient chart

➡️ Opens Carequality screen

---

### 2. System sends request to Surescripts

(Surescripts = network provider)

👉 Types of requests:

* **XCPD** → Find patient in other systems
* **XCA Query** → Check available documents
* **XDS Retrieve** → Download documents

---

### 3. Patient Matching (MPI)

* Your system sends patient details
* Surescripts checks **MPI (Master Patient Index)**

📌 Important:
👉 Your patient data must be uploaded first (`/uploadMPI`)

---

### 4. External Organizations Respond

* Other hospitals send:

  * Documents
  * Reports

---

### 5. Data Displayed in UI

* You see:
  ✔ List of documents
  ✔ Organization source

---

### 6. View / Download Documents

* Click **View**
* Download as **PDF**

---

# 🧩 Key Components (Simple View)

## 📁 Tables (Database)

* `carequality_patient_document_details` → stored docs
* `carequality_transaction_log` → request logs
* `patient`, `mpi` → patient matching

---

## ⚙️ Backend Services

* `CareQualityController`
* `CareQualityServiceImpl`

👉 Handle:

* Requests
* Responses
* Data processing

---

## 🌐 External Integration

* **Surescripts (Carequality network)**

👉 Communication via:

* SOAP APIs
* SFTP

---

# 🔁 Types of Requests (Very Important)

| Request Type           | Purpose         |
| ---------------------- | --------------- |
| **XCPD**               | Find patient    |
| **XCA Query**          | Check documents |
| **XCA Retrieve (XDS)** | Get documents   |

---

# 🧪 What You Do (As Developer / Tester)

## ✅ Testing Steps

1. Check CQ icon enabled
2. Open CQ page
3. Verify:

   * Data loading
   * Documents listed
4. Click **View**
5. Download PDF

---

## 🔧 Debug / Development Tasks

👉 You may work on:

### 1. API Issues

* `/uploadMPI`
* `/checkFTPResponse`

---

### 2. Data Issues

* Patient not matching (MPI issue)
* Documents not coming

---

### 3. UI Issues

* CQ page not loading
* “Failed to load” errors

---

### 4. Integration Issues

* Surescripts connection
* Certificate / SFTP issues

---

# 🧠 Simple Real-Time Example

👉 Patient visits your hospital
👉 You click **Carequality**

➡️ System:

1. Searches patient in other hospitals
2. Finds records
3. Fetches reports

👉 You can now see:

* Previous diagnosis
* Lab results
* Treatment history

---

# 💡 One-Line Summary

👉 **Carequality module = Fetch & share patient medical data from other hospitals securely**

---

# 🚀 Easy Way to Remember

👉 **CQ = Cross-hospital data sharing system**
