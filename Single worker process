# Process of each row file

```text
                                ┌───────────────────────────┐
                                │      ROW (JSON)           │
                                │   (get from worker)       │
                                └────────────┬──────────────┘
                                             │
                                             ▼
                          ┌──────────────────────────────────┐
                          │    STEP 1: key NORMALIZER        │
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
                          │   STEP 3: model create or update │
                          └────┬────────────────────────┬────┘
                               │                        │
    [Atomic Operation pp]      │  - try to find the model by this keys
                               │    {vc_code, model, variant, color, year}
                               │  - Action: findOne → Update OR Create

                               ▼
                  ┌──────────────────────────────────────────────┐
                  │  STEP 4: sub taskes which required mode_id   │
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
          (Offer error ≠ Job failure)              to the errors array (max 1000)

                               │
                               ▼
                    ┌───────────────────────────┐
                    │       DB COMMITMENT       │
                    │ - Seq (ORM) transactions  │
                    │ - Real-time progress %    │
                    └───────────────────────────┘
```
