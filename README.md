# IBM CDC Notebook (Databricks)

This repo contains a Databricks notebook that is intended to run on a **schedule** to pull IBM daily stock data from an **API** and process changes through a **layered (Bronze → Silver)** Delta Lake design.

This is **not** a Databricks Auto Loader pipeline/job. The notebook uses direct API ingestion and incremental logic to avoid extra Auto Loader costs.

Notebook:
- `IBM  - CDC.ipynb`
- Source: https://github.com/Ranoush75/Databricks_IBM/blob/first_upload/IBM%20%20-%20CDC.ipynb

---

## What it does

### Bronze layer (workspace.bronze.ibm) — ingestion + history
- Connects to the API and ingests daily IBM stock data (JSON → tabular).
- **Appends only new records** each scheduled run (based on a watermark / max `Date` in the existing Bronze table).
- **Keeps all historical data** (Bronze is the long-term, append-friendly history layer).

**Key idea:** Bronze grows over time and preserves what was ingested each run (historic retention).

### Silver layer (workspace.silver.ibm) — curated “latest version” per record
- Reads from the Bronze table.
- Applies cleaning/validation steps (casting types, dropping nulls on critical columns, de-duplication on the key).
- Loads into Silver using a Delta **MERGE (upsert)** keyed by `Date`.

**Key idea:** Silver represents the **latest version of each record (per `Date`)**:
- If a `Date` already exists → it can be updated (latest values kept)
- If a `Date` is new → it is inserted

---

## How it is intended to run (scheduled job)
This notebook is designed to be executed as a **scheduled Databricks Job** (e.g., daily). Each run:
1. Pulls the latest available data from the API
2. Appends new records into **Bronze** while keeping all history
3. Produces/updates **Silver** to reflect the latest version per `Date`

> Scheduling (cron/timezone, cluster, retries, alerts) is configured in Databricks Jobs, not inside the notebook.

---

## Requirements
- Databricks runtime with Delta Lake support
- Permissions to create/read/write:
  - `workspace.bronze.ibm`
  - `workspace.silver.ibm`

Python libraries used include:
- `requests`, `pandas`, `numpy`, `json`
- `pyspark` + `delta.tables`

---

## Tables

### Bronze
- `workspace.bronze.ibm` (Delta)
  - Append-only ingestion behavior
  - Historical retention

### Silver
- `workspace.silver.ibm` (Delta)
  - Cleaned/curated output
  - Latest version per key (`Date`) via MERGE/upsert

---

## Manual run steps
1. Open `IBM  - CDC.ipynb` in Databricks.
2. Attach to a cluster.
3. (Recommended) Use Databricks secrets for the API key.
4. Run the notebook top-to-bottom.

---

## Notes
- The notebook avoids Auto Loader to reduce cost.
- The merge key is `Date` (unique key for daily stock records). If that changes, update the merge condition accordingly.