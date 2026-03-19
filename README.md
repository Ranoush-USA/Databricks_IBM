

IBM - CDC Notebook
This is a Change Data Capture (CDC) pipeline notebook in Databricks for IBM stock data, built in two layers following the medallion architecture.

What CDC Means Here
CDC (Change Data Capture) means the notebook doesn't reload all data every run — it detects and ingests only new records since the last run, keeping both Bronze and Silver tables up to date incrementally.

Layer 1 — Bronze (Raw Ingestion)
Cell 2 — Imports
Loads requests, pandas, psycopg2, and standard Python libraries needed for API calls and data handling.
Cell 5 — API Fetch Function
Defines fetch_ibm_daily_data() which hits the Alpha Vantage TIME_SERIES_DAILY endpoint for IBM stock. It extracts Date, Open, High, Low, Close, Volume, and Last_updated from the JSON response and returns a Pandas DataFrame.
Cell 6 — Convert to Spark
Converts the Pandas DataFrame to a Spark DataFrame with spark.createDataFrame().
Cell 8 — Schema Validation
Compares the schema of the incoming spark_df against the existing workspace.bronze.ibm table to catch any column type mismatches before writing.
Cell 9 — Type Casting
Casts all columns to correct types — Date and Last_updated to DateType, price columns to DoubleType, and Volume to LongType since Alpha Vantage returns everything as strings.
Cell 11 — Watermark Filter (CDC Logic)
Gets the MAX(Last_updated) already stored in bronze.ibm, then filters spark_df to keep only rows newer than that value. This is the core CDC mechanism — only new data moves forward.
Cell 14 — Append to Bronze
Writes the filtered new rows to workspace.bronze.ibm using .mode("append").
Cell 18 — Verify Latest Write
Uses Delta time travel to read only the rows added in the latest version of the table, confirming the append worked correctly.

Layer 2 — Silver (Cleaning & Merge)
Cell 23 — Read Bronze
Reads workspace.bronze.ibm selecting only the 7 core columns, dropping any internal Auto Loader columns.
Cell 26 — Create Silver Table
Creates workspace.silver.ibm as a Delta table if it doesn't exist yet, using WHERE 1=0 to copy the schema from Bronze without copying any data.
Cell 29 — Full Cleaning Pipeline
Applies four cleaning steps before merging:

dropna() on critical columns to remove rows with missing prices or dates
dropDuplicates(["Date"]) to ensure one row per trading day
Type casting to enforce correct data types in Silver
.drop("_rescued_data") to remove the Auto Loader internal column

Cell 30 — Schema Check
Prints and compares the schemas of both the target Silver table and the cleaned source DataFrame to confirm they match before the merge runs.
Cell 31 — Merge into Silver (CDC)
Runs a Delta MERGE using Date as the unique key:

whenMatchedUpdateAll() — if a date already exists in Silver, update it with the latest values from Bronze
whenNotMatchedInsertAll() — if the date is new, insert it

Cell 33 — SQL Equivalent
Shows the plain SQL MERGE INTO statement that does the same thing as the Python merge above, useful for reference or running directly in a SQL warehouse.

Data Flow Summary
Alpha Vantage API
       ↓
fetch_ibm_daily_data()        ← pulls raw JSON
       ↓
Type casting + schema check   ← validates before write
       ↓
Watermark filter              ← CDC: only new rows
       ↓
workspace.bronze.ibm          ← append new rows only
       ↓
Drop nulls + dedup + cast     ← Silver cleaning
       ↓
workspace.silver.ibm          ← MERGE (upsert by Date)

Key Design Decisions
DecisionReasonWatermark on Last_updatedLightweight CDC without full table scanDate as merge keyNatural unique key — one row per trading daywhenMatchedUpdateAll in SilverHandles corrected prices from sourceDROP _rescued_data before mergeAuto Loader column not needed in SilverSchema check before mergeCatches type mismatches early before errors