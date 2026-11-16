
# 🚀 MIPS Kafka Batch Processing

> Documentation for running MIPS batch processing using Java, Apache Kafka, PostgreSQL, and multiple Worker instances.

## 📌 Project Overview

The MIPS Kafka Batch Application is designed to process multiple accounts concurrently.

It uses:

- **Java 17** – Application runtime
- **Apache Kafka** – Account-level job distribution
- **PostgreSQL** – Batch tracking, account status, patient progress, and failure records
- **ExecutorService** – Patient-level parallel processing
- **HikariCP** – Database connection pooling

### Architecture

```text
                  accounts.csv
                       |
                       v
                  MASTER PROCESS
                       |
                       v
                 Kafka Producer
                       |
                       v
               mips-account-jobs
                  (3 partitions)
                       |
             +---------+---------+
             |                   |
             v                   v
         WORKER 1             WORKER 2
             |                   |
             v                   v
        Account Job          Account Job
             |                   |
             v                   v
      Patient Threads      Patient Threads
       (10 threads)         (10 threads)
             |                   |
             v                   v
       GlaceEMR APIs        GlaceEMR APIs
             |                   |
             +---------+---------+
                       |
                       v
                  PostgreSQL
                       |
                       v
                 Batch Status
```

### Processing Flow

1. Master reads account codes from `config/accounts.csv`.
2. Master creates a new batch and account job records.
3. Master publishes one Kafka message per account.
4. Kafka distributes messages among available Workers.
5. Each Worker processes its assigned account.
6. Each Worker uses patient-level threads to process patients concurrently.
7. Failed patients are retried according to the configured retry settings.
8. After successful patient processing, the final MIPS calculation runs.
9. PostgreSQL stores account status and processing progress.

---

## 📂 Project Location

```bash
cd ~/Projects/mips-kafka-batch
```

## ⚙️ 1. Environment Setup

Use these commands in **every new terminal**.

```bash
cd ~/Projects/mips-kafka-batch

export JAVA_HOME=/usr/lib/jvm/java-17-openjdk-amd64
export PATH="$JAVA_HOME/bin:$PATH"

set -a
source .env
set +a

export MIPS_DB_PASSWORD="$POSTGRES_PASSWORD"
```

Verify Java:

```bash
java -version
```

Expected runtime: Java 17.

## 🐳 2. Start Kafka and PostgreSQL

```bash
docker compose up -d
```

Check containers:

```bash
docker ps
```

Expected containers:

- `mips-kafka`
- `mips-postgres`

Check Kafka topic:

```bash
docker exec mips-kafka /opt/kafka/bin/kafka-topics.sh \
  --bootstrap-server localhost:19092 \
  --describe \
  --topic mips-account-jobs
```

Current configuration:

- Topic: `mips-account-jobs`
- Partitions: 3
- Replication factor: 1
- Consumer group: `mips-worker-group`

## 📋 3. Configure Account List

Edit the CSV:

```bash
nano config/accounts.csv
```

Example:

```csv
account_name,reporting_year
vhampole,2026
tmehra,2026
```

**Important:**

- Add only the accounts you want to process in this batch.
- Use the correct account codes.
- Set the required reporting year.
- Master creates a new batch using all rows in this CSV.
- Do not include previously processed accounts unless you intentionally want to process them again.

## 👷 4. Start Multiple Workers

Each Worker must run in a separate terminal.

All Workers use the same Kafka consumer group:

```text
mips-worker-group
```

### Terminal 1 – Start Worker 1

First, perform the environment setup from Section 1.

Then execute:

```bash
MIPS_WORKER_ID=worker-1 nohup java \
  -Dmips.threads=10 \
  -Dmips.chunk=8 \
  -Dmips.delayMs=1000 \
  -jar target/MIPSBatchApplication.jar worker \
  > logs/worker-1-console.log 2>&1 &

echo "Worker 1 PID: $!"
```

### Terminal 2 – Start Worker 2

Perform the environment setup again in Terminal 2.

Then execute:

```bash
MIPS_WORKER_ID=worker-2 nohup java \
  -Dmips.threads=10 \
  -Dmips.chunk=8 \
  -Dmips.delayMs=1000 \
  -jar target/MIPSBatchApplication.jar worker \
  > logs/worker-2-console.log 2>&1 &

echo "Worker 2 PID: $!"
```

### Verify Workers

```bash
pgrep -af 'MIPSBatchApplication.jar worker'
```

Check Kafka consumer assignments:

```bash
docker exec mips-kafka /opt/kafka/bin/kafka-consumer-groups.sh \
  --bootstrap-server localhost:19092 \
  --describe \
  --group mips-worker-group
```

**Note:** Kafka distributes partitions, not individual accounts directly. Two Workers do not guarantee that two account jobs will execute simultaneously. Account keys and partition assignments determine the actual distribution.

## 🚀 5. Run Master

After the Workers are running and connected to Kafka, open a terminal and perform the environment setup.

Run Master **only once for each new batch**:

```bash
java -jar target/MIPSBatchApplication.jar master
```

Master will:

1. Read `config/accounts.csv`.
2. Create a new batch UUID.
3. Insert account job records into PostgreSQL.
4. Publish account jobs to Kafka.

**Save the Batch UUID** printed by Master. It is required to monitor the batch.

Do not run Master again just to restart a Worker or check progress. Running Master creates another batch.

## 📊 6. Monitor Batch Status

Use the Batch UUID generated by Master:

```bash
java -jar target/MIPSBatchApplication.jar status YOUR_BATCH_UUID
```

Example:

```bash
java -jar target/MIPSBatchApplication.jar status 5d9d7fb1-f559-434c-8bdc-03ce7d793c05
```

The status output displays:

| Field | Description |
|---|---|
| ACCOUNT | Account code |
| STATUS | Current account status |
| SELECTED | Total selected patients |
| PROCESSED | Patients recorded in the processing ledger |
| SUCCESS | Successfully processed patients |
| FAILED | Patients not resolved successfully |
| REMAINING | Patients not yet recorded |
| FINAL_CALC | Final MIPS calculation status |

### PostgreSQL Account Status

```sql
SELECT
    account_code,
    status,
    total_patients,
    processed_patients,
    successful_patients,
    failed_patients,
    final_calculation_status,
    last_error
FROM mips_job_accounts
ORDER BY account_code;
```

For a specific batch:

```sql
SELECT
    account_code,
    status,
    total_patients,
    processed_patients,
    successful_patients,
    failed_patients,
    final_calculation_status,
    last_error
FROM mips_job_accounts
WHERE job_id = 'YOUR_BATCH_UUID'
ORDER BY account_code;
```

## 📝 7. Monitor Application Logs

### Worker 1

```bash
tail -f logs/worker-1-console.log
```

### Worker 2

```bash
tail -f logs/worker-2-console.log
```

### Main Application Log

```bash
tail -f logs/mips-batch.log
```

### View Latest 100 Lines

```bash
tail -n 100 logs/mips-batch.log
```

### Account Processing Summary

```bash
grep -E 'Account:|Fetching patient list|Total patients|Patient processing completed' \
logs/mips-batch.log
```

### Patient Progress

```bash
grep 'Progress:' logs/mips-batch.log | tail -n 30
```

### Errors and Retries

```bash
grep -Ei 'ERROR|WARN|exception|retrying failed' \
logs/mips-batch.log | tail -n 80
```

### Final MIPS Calculation

```bash
grep -Ei 'Final MIPS calculation|Account .* completed' \
logs/mips-batch.log
```

## 🔁 8. Adding More Workers

The current Kafka topic has 3 partitions.

To increase account-level parallelism:

1. Check the current Kafka partition count.
2. Increase partitions when necessary.
3. Start additional Worker JVMs.
4. Keep the same Kafka consumer group.
5. Verify partition assignments.
6. Publish a new batch using Master.

Example: Start Worker 3:

```bash
MIPS_WORKER_ID=worker-3 nohup java \
  -Dmips.threads=10 \
  -Dmips.chunk=8 \
  -Dmips.delayMs=1000 \
  -jar target/MIPSBatchApplication.jar worker \
  > logs/worker-3-console.log 2>&1 &
```

**Important considerations:**

- Kafka partitions limit the number of Workers that can actively consume in the same group.
- Increasing partitions does not guarantee even account distribution.
- Each Worker has its own patient thread pool and per-host concurrency limit.
- The current per-host concurrency setting is 8 per Worker, not a global limit.
- More Workers increase the total load on GlaceEMR APIs.
- Validate account leases, retry behaviour, and consumer rebalance safety before scaling.
- Get backend capacity approval before increasing concurrency.

## 🛑 9. Stop Workers

Check running processes:

```bash
pgrep -af 'MIPSBatchApplication.jar worker'
```

Stop a specific Worker using its PID:

```bash
kill WORKER_PID
```

Check whether it has stopped:

```bash
pgrep -af 'MIPSBatchApplication.jar worker'
```

Avoid using `kill -9` unless normal shutdown fails and the consequences have been checked.

Do not stop a Worker in the middle of account processing without checking its account lease, in-flight requests, and Kafka offset handling.

## 🛠️ 10. Build the Application

After source code changes:

```bash
export JAVA_HOME=/usr/lib/jvm/java-17-openjdk-amd64
export PATH="$JAVA_HOME/bin:$PATH"

mvn clean package -DskipTests
```

Generated JAR:

```text
target/MIPSBatchApplication.jar
```

## 📌 11. Important Operational Rules

1. Start Kafka and PostgreSQL before starting Workers.
2. Start the required Workers before running Master.
3. Run Master only once per new batch.
4. Save the batch UUID.
5. Keep all Workers in the same Kafka consumer group.
6. Verify Kafka partition distribution before assuming parallel account execution.
7. Monitor patient success and failure counts.
8. Check the final MIPS calculation status separately.
9. Do not treat PROCESSED as SUCCESS.
10. Do not rerun an account blindly after a timeout; the API may already have processed the request.
11. Never delete PostgreSQL volumes or batch records just to restart processing.
12. Use the correct account list before publishing a new batch.

---

## 📈 Previous Test Run – November 16, 2026

### Test Accounts

| Account | Total Patients | Successful | Failed | Final Calculation |
|---|---:|---:|---:|---|
| GERB | 1,115 | 1,115 | 0 | SUCCEEDED |
| MAKP | 1,099 | 1,099 | 0 | SUCCEEDED |
| ZMC | 1,122 | 1,121 | 1 | NOT_STARTED |

### Test Observations

- GERB processed all 1,115 patients successfully.
- MAKP processed all 1,099 patients successfully.
- ZMC had one unresolved patient.
- The existing application skipped final MIPS calculation for ZMC because its completion check required all patients to succeed.
- The batch demonstrated patient-level parallel processing and retry handling.

### Next Development Goals

- Support partial patient failures without unnecessarily blocking final MIPS calculation, subject to business approval.
- Improve patient-level failure diagnostics.
- Validate multi-Worker account-level parallel processing.
- Improve batch progress monitoring.
- Evaluate performance with a larger number of accounts.
- Add safe recovery and retry handling.

---

**Project:** MIPS Kafka Batch Processing  
**Runtime:** Java 17  
**Message Broker:** Apache Kafka  
**Database:** PostgreSQL  
**Processing Model:** Account-level Kafka Workers + Patient-level ExecutorService
