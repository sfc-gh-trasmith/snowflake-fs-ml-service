# Feature Store + ML Inference Service Demo

End-to-end demo of Snowflake's **Online Feature Store (Postgres-backed)** integrated with a real-time **ML Inference Service** using `feature_sources_per_function`.

## Why This Matters

In production ML systems, one of the hardest problems is **training-serving skew** — when the features used at inference time drift from those used during training. This happens because training pipelines and serving pipelines are often built separately, with different code paths, different data sources, and different transformation logic.

The **Feature Store + Inference Service integration** solves this by:

- **Single source of truth** — The same Feature View definitions serve both training and inference. No duplicate feature logic to maintain.
- **Decoupled clients from feature complexity** — API consumers send only an entity key (e.g. a user ID). They don't need to know which features exist, how they're computed, or where they're stored.
- **Low-latency serving** — The Postgres-backed online store provides millisecond-level feature retrieval, purpose-built for real-time inference workloads.
- **Zero-effort feature freshness** — Features sync automatically from the source table to the online store with a configurable target lag (e.g. 10 seconds). No ETL pipelines to manage.
- **Simplified service contracts** — When you add, remove, or modify features, the inference service adapts without requiring API changes or client updates.

This pattern is essential for any real-time ML application — fraud detection, recommendations, dynamic pricing, personalization — where you need fresh features served at low latency without burdening the caller with feature assembly.

## What This Notebook Does

This notebook uses **synthetic data** to demonstrate a customer churn ML example. The same pattern applies to any ML use case where you want low-latency feature serving integrated with real-time inference.

1. Creates a synthetic dataset (200 records)
2. Sets up a **Feature Store** with an entity and Feature View
3. Enables **Postgres-backed online serving** with 10-second target lag
4. Trains an **XGBoost** classifier
5. Registers the model in the **Snowflake Model Registry**
6. Deploys a real-time **inference service** on SPCS with `feature_sources_per_function`
7. Shows how to invoke the service — callers send only the entity key and the service fetches features automatically

## Architecture

```
External Client                Snowflake Ingress Gateway              SPCS Model Container
     |                                  |                                      |
     |--- POST /predict ------------->  |                                      |
     |    {"ENTITY_KEY": "id_123"}      |                                      |
     |                                  |--- fetch features from FS -------->  |
     |                                  |    (Postgres online store)            |
     |                                  |                                      |
     |                                  |--- enriched payload --------------> |
     |                                  |    {feature_1, feature_2, ...}       |
     |                                  |                                      |
     |<--- prediction ---------------  |<--- model prediction --------------- |
     |    {"output": 0}                 |                                      |
```

## Prerequisites

- Snowflake account with **ACCOUNTADMIN** role
- A compute pool (the notebook uses `ML_ONLINE_CPU_POOL`)
- Snowflake Notebook with **Container Runtime** (CPU)
- `snowflake-ml-python >= 1.44.0`

## Quickstart

### 1. Open the Notebook

Open `ml_service_fs.ipynb` in a Snowflake Workspace with a Container Runtime service (CPU, Python 3.10+).

### 2. Run All Cells in Order

The notebook is self-contained. Run all cells top-to-bottom:

- **Part 1** sets up the Feature Store with Postgres online serving
- **Part 2** trains, registers, and deploys the model

### 3. Set Up a PAT (Programmatic Access Token)

The online Feature Store and inference service require a PAT for authentication:

```sql
ALTER USER <your_user> ADD PROGRAMMATIC ACCESS TOKEN MY_PAT COMMENT = 'For inference service';
```

Store it as a Snowflake secret:

```sql
CREATE SECRET <db>.<schema>.MY_PAT_SECRET
  TYPE = GENERIC_STRING
  SECRET_STRING = '<your_pat_token>';
```

The notebook expects `SNOWFLAKE_PAT` in the environment. Configure it via notebook **Settings > Secrets**.

### 4. Invoke the Service

From any external client (curl, Python, web app):

```bash
curl -X POST "https://<ENDPOINT_URL>/predict" \
  -H 'Authorization: Snowflake Token="<PAT_TOKEN>"' \
  -H 'Content-Type: application/json' \
  -d '{"dataframe_split": {"index": [0, 1, 2], "columns": ["CUSTOMER_ID"], "data": [["CUST_0001"], ["CUST_0010"], ["CUST_0050"]]}}'
```

You send **only** `CUSTOMER_ID` — the service fetches all features from the Postgres online store automatically via `feature_sources_per_function`.

## Cleanup

```sql
DROP SERVICE IF EXISTS ML_DEMO_INF_FS_LOOKUP.ML_SERVICE_FS.CHURN_PREDICTION_SVC;
DROP MODEL IF EXISTS ML_DEMO_INF_FS_LOOKUP.ML_SERVICE_FS.CHURN_XGBOOST;
DROP DATABASE IF EXISTS ML_DEMO_INF_FS_LOOKUP;
DROP EXTERNAL ACCESS INTEGRATION IF EXISTS CHURN_SVC_EAI;
```

## References

- https://docs.snowflake.com/en/developer-guide/snowflake-ml/inference/real-time-inference-rest-api
