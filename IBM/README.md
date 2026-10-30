
## IBM_ETL Job - Complete Workflow Description

This is a **3-stage sequential ETL pipeline** for processing IBM stock market data with email notifications and performance optimization.

### **Job Configuration**
* **Name:** IBM_ETL
* **Execution Mode:** Queue-enabled (allows multiple runs to queue if previous run is still active)
* **Performance Target:** PERFORMANCE_OPTIMIZED (uses faster compute resources)
* **Notifications:** Sends emails to rana@ghazzi.com on both success and failure

---

### **Task 1: API-Ingestion**
**Notebook:** IBM_landing  
**Dependencies:** None (first task)

**What it does:**
1. Loads configuration from `config_Parms` (catalog, schemas, API key)
2. Calls Alpha Vantage API to fetch IBM daily stock data (Open, High, Low, Close, Volume)
3. Transforms API JSON response into pandas DataFrame
4. **Incremental Logic:** Queries existing `workspace.bronze.ibm` table to find the latest date
5. Filters only NEW records (dates newer than what exists in bronze)
6. Writes new records as **Parquet files** to landing zone: `/Volumes/workspace/bronze/landing_zone/ibm_landing/`

**Output:** Parquet files ready for streaming ingestion

---

### **Task 2: Auto_Loader_bronze**
**Notebook:** IBM_autoLoader_bronze  
**Dependencies:** Waits for API-Ingestion to complete

**What it does:**
1. Loads configuration from `config_Parms`
2. Uses **Auto Loader (cloudFiles)** to automatically detect new Parquet files in landing zone
3. Streams data with:
   - Schema location: `/Volumes/workspace/bronze/schemas/ibm_stream`
   - Checkpoint location: `/Volumes/workspace/bronze/checkpoints/ibm_stream`
   - Merge schema enabled (handles schema evolution)
4. Writes streaming data to **Delta table:** `workspace.bronze.ibm`
5. Uses `trigger(availableNow=True)` for micro-batch processing (processes all available data then stops)

**Output:** Raw data in bronze Delta table with exactly-once processing semantics

---

### **Task 3: Silver_Merge**
**Notebook:** IBM _silver  
**Dependencies:** Waits for Auto_Loader_bronze to complete

**What it does:**
1. Loads configuration from `config_Parms`
2. Creates silver table schema if it doesn't exist
3. Reads from `workspace.bronze.ibm`
4. **Data Quality Transformations:**
   - Casts string columns to proper types (DATE, DOUBLE, BIGINT, TIMESTAMP)
   - Drops null values on critical columns (Date, Open, High, Low, Close, Volume)
   - Deduplicates by Date (keeps latest record per date)
   - Filters invalid data (Close > 0)
   - Adds `ingested_at` timestamp
5. **MERGE operation** into `workspace.silver.ibm`:
   - If Date exists: UPDATE all columns
   - If Date is new: INSERT new row
6. Result: Clean, typed, deduplicated data ready for analytics

**Output:** Business-ready data in silver Delta table

---

### **Data Flow Summary**
```
Alpha Vantage API 
  ↓
Landing Zone (Parquet)
  ↓
Bronze Layer (Delta, raw strings)
  ↓
Silver Layer (Delta, typed & cleaned)
```

### **Key Features**
* **Idempotent:** Can rerun safely without duplicates (merge on Date key)
* **Incremental:** Only processes new data based on date comparison
* **Fault-tolerant:** Checkpointing ensures no data loss
* **Schema-aware:** Auto Loader handles schema changes automatically

View all notebooks
