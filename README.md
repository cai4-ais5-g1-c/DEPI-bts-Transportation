BTS Flight On-Time Performance – Azure End-to-End Data Pipeline

Project Overview

This project builds an **end-to-end data pipeline** on Microsoft Azure to analyze **airline on-time performance** using public data from the **Bureau of Transportation Statistics (BTS)**.  
The pipeline ingests raw flight data, transforms it into a star schema, and enables analytical queries for delays, cancellations, and trends.

**Key technologies:**  
- Azure Data Lake Storage Gen2 (raw & curated zones)  
- Azure Synapse Analytics (Dedicated SQL Pool)  
- Azure Data Factory / Synapse Pipelines (orchestration)  
- T-SQL (dimensional modeling, fact tables)  
- GitHub (version control) + GitHub Pages (documentation site)

---

🗂️ Data Model (Star Schema)

Dimension Tables
- `dim_date` – date attributes (year, quarter, month, weekend, leap year, etc.)  
- `dim_airport` – origin/destination airport codes and details  
- `dim_airline` – airline codes and names  
- `dim_aircraft` – aircraft type and configuration (optional)

Fact Table
- `fact_flights` – one row per flight with measures:  
  - Scheduled/actual departure & arrival times  
  - Delay minutes (departure, arrival, carrier, weather, NAS, security, late aircraft)  
  - Cancellation/divert indicators  

*Distribution:** `HASH([FlightKey])` – `CLUSTERED COLUMNSTORE INDEX` for performance.

---

## 🚀 Pipeline Architecture


| Step | Component | Description |
|------|-----------|-------------|
| **1. Ingestion** | Azure Data Factory / Synapse Pipeline | Copies raw BTS `.csv` files from public HTTP source or uploaded blob to **Bronze container** (ADLS Gen2). |
| **2. Transformation** | Mapping Data Flow | Cleans data: handles nulls, casts types, derives delay categories, adds `SurrogateKey` for fact table. Outputs **Silver container** as Parquet files. |
| **3. Loading** | `COPY INTO` or PolyBase | Loads Silver Parquet into staging tables in Synapse SQL Pool. |
| **4. SCD & Modelling** | Stored Procedure (`usp_LoadFactFlights`) | Updates dimensions (Type 1/2 if needed) and inserts into `fact_flights` with referential integrity checks. |
| **5. Scheduling** | Trigger (Tumbling Window) | Runs daily at 02:00 UTC to process previous day’s flight data. |
| **6. Monitoring** | Azure Monitor + Log Analytics | Alerts on pipeline failures, data skew, or row count mismatches. |
| **7. Serving** | Power BI DirectQuery | Connects to Synapse SQL Pool for real‑time dashboards. |

### Distribution & indexing strategy (Synapse Dedicated SQL Pool)

- **Small dimensions** (`dim_date`, `dim_airline`): `DISTRIBUTION = REPLICATE` + `CLUSTERED INDEX`.
- **Large fact table** (`fact_flights`): `DISTRIBUTION = HASH([FlightKey])` + `CLUSTERED COLUMNSTORE INDEX` (compression & scan performance).

### Why this design?

- **Scalability** – MPP engine handles billions of flight records.  
- **Cost control** – Bronze/Silver stay in cheap ADLS; Synapse pool paused when not in use.  
- **Reproducibility** – Full pipeline as code (ARM / JSON) in this repo.

> 📌 **Note:** The pipeline avoids expression‑based `DEFAULT` constraints in the fact table – timestamps are populated via `GETDATE()` inside the `INSERT` logic.
