# 🚀 CSV Upload & Processing System

A scalable, fault-tolerant CSV ingestion pipeline designed to handle large datasets with high accuracy, modular processing, and real-time progress tracking.

---

## 🧠 System Overview

This system is designed to:

* Process large CSV files efficiently using streaming & batching
* Ensure data consistency with atomic DB operations
* Support modular business logic (Vehicle, Insurance, VAS, Offers)
* Provide real-time progress updates
* Maintain fault tolerance with retry mechanisms and row-level isolation

---

## 🔄 Row Processing Pipeline (Detailed Flow)

```
                                ┌───────────────────────────┐
                                │      ROW (JSON)           │
                                │   (get from worker)       │
                                └────────────┬──────────────┘
                                             │
                                             ▼
                          ┌──────────────────────────────────┐
                          │    STEP 1: KEY NORMALIZER        │
                          └────┬────────────────────────┬────┘
                               │                        │
         [Header Mapping]      │  - Maps "VC Code" → "vc_code"
                               │  - Maps "branch_1_(Hub)" → "branch_1"
                               │  - Maps "VAS_Name1" → "vas_name1"

                               ▼
                          ┌──────────────────────────────────┐
                          │    STEP 2: SECTION DETECTION     │
                          │     (vas, insurance etc)         │
                          └────┬────────────────────────┬────┘
                               │                        │
        [Module Triggering]    │  - if 'vc_code'     → Set hasVehicle
                               │  - if 'offer1'      → Set hasOffers
                               │  - if 'insurance1'  → Set hasInsurance

                               ▼
                          ┌──────────────────────────────────┐
                          │   STEP 3: MODEL CREATE/UPDATE    │
                          └────┬────────────────────────┬────┘
                               │                        │
    [Atomic Operation pp]      │  - try to find model by:
                               │    {vc_code, model, variant, color, year}
                               │  - Action: findOne → Update OR Create

                               ▼
                  ┌──────────────────────────────────────────────┐
                  │  STEP 4: SUBTASKS (REQUIRE model_id)         │
                  └──────┬───────────────┬───────────────────────┘
                         │               │
      ┌──────────────────┴───────────────┼───────────────────────────────┐
      │                  │               │                               │
      ▼                  ▼               ▼                               ▼
  INSURANCE          ACCESSORIES          VAS                           OFFERS

- findOrCreate      - findOrCreate Acc - findOrCreate VAS            - STRICT FIND Offer:
  (LOWER match)       (master table)     (master table)               **NO auto-create**
- upsert Base Price - upsert Model link- upsert Model link           - If missing: Log Error
- upsert Addon      - store model-spec - parse duration              - upsert Model link
  (ModelInsAddon)     price                                          - status: visible=true

      │                  │               │                               │
      └──────────┬───────┴───────────────┴───────────────┬───────────────┘
                 │                                       │
                 ▼                                       ▼
        [ACCURACY AUDIT]                         [FINAL CONSOLIDATION]
        - Stats: {inserted, updated}             - return stats object
        - Row-level isolation:                   - push parsing/logic errors
          (Offer error ≠ Job failure)              to errors array (max 1000)

                               │
                               ▼
                    ┌───────────────────────────┐
                    │       DB COMMITMENT       │
                    │ - Seq (ORM) transactions  │
                    │ - Real-time progress %    │
                    └───────────────────────────┘
```

---

## 📁 File Upload Architecture (End-to-End Flow)

```
                                ┌───────────────────────────┐
                                │        FRONTEND           │
                                │    (CsvUploader.jsx)      │
                                └────────────┬──────────────┘
                                             │
                     ┌───────────────────────┼────────────────────────┐
                     │                       │                        │
                     ▼                       ▼                        ▼

        [1] User selects file     [2] Upload in parts       [3] Start process
        GET /presigned-url        POST /proxy-upload        POST /start

                     │                       │                        │
                     └──────────────┬────────┴──────────────┬─────────┘
                                    ▼                       ▼

                          ┌───────────────────────────┐
                          │        BACKEND API        │
                          │   (import.controller)     │
                          └────────────┬──────────────┘
                                       │
        ┌──────────────────────────────┼──────────────────────────────┐
        │                              │                              │
        ▼                              ▼                              ▼

   [File Row Logic]          [System Check]              [creat job Send to Queue]
   - count total parts       - check Redis               - queue: csv-import
   - create uploadId         - check DB                  - send:
                             - check worker                { jobId, fileKey }

  =>
┌───────────────────────────┐
│      REDIS (BullMQ)       │
│  - job queue              │
│  - retry if failed (3x)   │
│  - track progress         │
└────────────┬──────────────┘
             │
             ▼

┌───────────────────────────┐
│         WORKER            │
│   (Background Processor)  │
└────────────┬──────────────┘
             │
   ┌─────────┼───────────────────────────────────────────────┐
   │         │                                               │
   ▼         ▼                                               ▼

 [Get File]   [Read + Detect]                      [Process in Batches]
 - stream file  - read headers                     for each batch:
 - don’t load   - detect sections:                 - clean data
   full file      Vehicles / Offers / VAS          - validate data
                                                     - insert/update DB

   │
   ▼

[Save to Database]
┌───────────────────────────┐
│         DATABASE          │
│  - insert or update       │
│  - avoid duplicates       │
└────────────┬──────────────┘
             │
             ▼

[Progress Update]
- count success / fail
- calculate %
- store in Redis
- UI via WebSocket
```

---

## ⚙️ Deployment

```
REDIS_HOST=127.0.0.1
REDIS_PORT=6379
```

```
pm2 start src/server.js --name "quotation-api"
pm2 start index.js --name "quotation-worker"
```

---

## 💡 Key Highlights

* Scalable architecture for large CSV ingestion
* Memory-efficient streaming (no full file load)
* Retry mechanism (3x) using BullMQ
* Strong data consistency with transactional operations
* Modular processing (Insurance, VAS, Offers, Accessories)
* Real-time progress tracking via WebSocket
* Robust error isolation (row-level fault tolerance)
