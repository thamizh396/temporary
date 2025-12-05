


Logic for claiming the thread based failure or claim:

While assigning the partitions, each thread will have some certain batch start and end. So for each thread had to set a key value with the thread name plus some identifier and the value will have the start and end
Eg : thread_1_startend = start:101,end:200

Then once the process got ended then had to claim/delete the cache key as the entries has been processed.

Here comes the query, once on the same process end or on the next job, we should pick these missed out records and reprocess the data without any duplication.


Logic for start up:

We have to get the last processed id or the next processed from the cache key as below,
Last_processed_id: 100
Next_processed_id: 101

So if the key are not present in DB on first run or if cache gets down had to fetch these ids from the db of the max transaction ID of output or error table. Then consider this as the last processed id.

Then we have to


Output table - will contain the records which got successfully validated.

error table - will contain the records which got failed while validation

Source table - will contain the records  


# **📄 High-Level Design Document — Multi-Threaded Processing Using Redis Partitioning**

## **1. Overview**

This document describes the high-level design for enhancing an existing batch processing service by introducing **multi-threaded processing with Redis-based ID partitioning**.
The goal is to ensure **parallel execution**, **ordered ID consumption**, and **zero duplication**, without using Spring Batch or Quartz tables for tracking.

---

## **2. Key Objectives**

* Enable **parallel processing** using multiple threads.
* Guarantee that **each transaction ID (txnId) is processed exactly once**.
* **Avoid duplicates** even across restarts.
* Recover seamlessly even if a thread fails during processing.
* Use Redis as the **only source of truth** for next IDs and missed IDs.
* Avoid storing validation-failed IDs (these are handled separately in the existing error table).

---

## **3. Redis Cache Keys (Minimal & Sufficient)**

### **3.1 `GLOBAL:NEXT_ID`**

* Tracks the **next unprocessed txnId**.
* All threads use atomic Redis `INCRBY` to claim ID ranges.
* Ensures ordered, lock-free ID allocation.

### **3.2 `THREAD:{id}:FAILED_IDS`**

* Stores txnIds that were **not processed due to infra/db failure**.
* NOT used for validation errors.
* Cleared once reprocessed successfully.

### **3.3 `RUN:STATUS`**

* Values: `RUNNING` / `STOPPED`.
* Prevents accidental multiple runs.

---

## **4. Startup Logic**

When the service starts:

### **4.1 Initialize `GLOBAL:NEXT_ID`**

* If the key **exists**, continue from it.
* If the key **does not exist**, derive from DB:

```
maxOutputId = SELECT MAX(txn_id) FROM output_table
maxErrorId  = SELECT MAX(txn_id) FROM error_table (validation errors)
globalNextId = max(maxOutputId, maxErrorId) + 1
SET GLOBAL:NEXT_ID = globalNextId
```

**Rationale:**
We resume from the highest successfully or unsuccessfully (validation) processed record.

---

### **4.2 Load Failed IDs**

For each thread:

* Check `THREAD:{id}:FAILED_IDS`
* If present → these IDs must be reprocessed **before picking new IDs**.

---

### **4.3 Set Run Status**

```
SET RUN:STATUS RUNNING
```

---

## **5. Thread Processing Logic**

### **5.1 Partition Assignment**

Each thread claims ID ranges using:

```
startId = INCRBY GLOBAL:NEXT_ID rangeSize
endId = startId + rangeSize - 1
```

This ensures:

* No two threads overlap
* No manual locking
* Perfect linear progression

---

### **5.2 Processing Flow Per Thread**

1. Check if `FAILED_IDS` exists → process those IDs first.
2. Query source table for txnIds between `startId` and `endId`.
3. For each txnId:

   * Validate
   * Transform
   * Insert into output table
4. If any infra or DB failure happens:

   * Push that txnId to `THREAD:{id}:FAILED_IDS`
5. On success:

   * Move to next ID until end of range

---

## **6. Failure Handling**

### **6.1 Thread-Level Failure**

If a thread dies midway:

* Only unprocessed IDs in its range are lost temporarily.
* No duplicates happen because the range was already assigned.
* In the next run:

  * IDs missing in the output table reappear as gaps.
  * OR infra-failed IDs exist in `FAILED_IDS`.
* Next run automatically picks them up:

  * First FAILED_IDS
  * Then continue from GLOBAL:NEXT_ID

---

### **6.2 Redis Down Scenario**

If Redis becomes unavailable:

* The system cannot partition IDs → batch should stop.
* When Redis comes back:

  * Startup logic recomputes `GLOBAL:NEXT_ID`
  * System resumes from DB-derived max ID
* **No data loss** except non-persistent FAILED_IDS

  * Acceptable as per requirement

---

## **7. End-of-Run Logic**

Once all threads complete:

```
SET RUN:STATUS STOPPED
```

* No other cache updates needed.
* All future runs start cleanly from `GLOBAL:NEXT_ID`.

---

## **8. High-Level Flow Diagram**

```
                 ┌────────────────────────┐
                 │        Startup         │
                 └───────────┬────────────┘
                             │
                Check GLOBAL:NEXT_ID exists?
                             │
         ┌───────────────────┼───────────────────┐
         │ Exists → Continue │ Not Exists → Load │
         │                   │ from DB max IDs   │
         └───────────────────┴───────────────────┘
                             │
              Load THREAD:X:FAILED_IDS (if any)
                             │
                Launch Multi-thread Execution
                             │
        ┌────────────────────┼─────────────────────────┐
        │ Thread 1           │ Thread 2                 │
        │ INCRBY assigns     │ INCRBY assigns          │
        │ ID range           │ ID range                │
        └────────────────────┼─────────────────────────┘
                             │
                     Process Records
                             │
        ┌────────────────────┬─────────────────────────┐
        │ Success → output   │ Infrastructure Failure  │
        │ table insert       │ → store in FAILED_IDS   │
        └────────────────────┴─────────────────────────┘
                             │
                      All Threads Done
                             │
                    RUN:STATUS = STOPPED
```

---

## **9. Advantages of This Approach**

* Zero duplication or overlap
* No DB locking required
* Fast ID partition allocation via Redis
* Automatic recovery through:

  * DB max IDs (for gaps)
  * FAILED_IDS (for infra errors)
* Clean, minimal architecture
