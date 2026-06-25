## Interview-ready summary of this project

This project is a complete Azure data engineering solution built around Azure Data Factory. Its purpose is to ingest data from multiple sources, land it in Azure Data Lake Storage, transform it into cleaner business-ready datasets, and publish analytics-ready outputs. In simple terms, it is a mini end-to-end data platform on Azure.

### What this project is doing
- It pulls data from:
  - on-premises CSV files using a Self-Hosted Integration Runtime,
  - an API endpoint that returns JSON data,
  - an Azure SQL Database table.
- It moves the data into Azure Data Lake Storage Gen2.
- It transforms the data using Azure Data Factory Mapping Data Flows.
- It stores the transformed datasets in Delta format in the “silver” and “gold” layers.
- It orchestrates everything through a parent pipeline and uses a failure alert step at the end.

So the project follows a data pipeline pattern:
1. Ingest data
2. Clean and transform it
3. Serve it for reporting/analytics

---

## Azure components used

### 1. Azure Data Factory
This is the main orchestration engine. It is used to run:
- Copy activities
- Lookup activities
- ExecutePipeline activities
- Data Flow activities

### 2. Azure SQL Database
The source transactional data comes from Azure SQL. The project reads from a table called FactBookings.

### 3. Azure Data Lake Storage Gen2
This is the storage layer for the raw and processed data. The project uses ADLS Gen2 with folders like:
- silver
- gold

### 4. Self-Hosted Integration Runtime
This is important because the project is reading files from an on-prem location. Azure Data Factory cannot directly access on-prem file systems without a Self-Hosted Integration Runtime.

### 5. Delta Lake
The project uses Delta format in the data flows for the silver and gold layers. That is a modern lakehouse-style approach and is good for scalable analytics and incremental/upsert-style processing.

---

## End-to-end flow of the project

### A. On-prem ingestion
The first pipeline, onprem-ingestion.json, loops through a list of files and copies them from an on-prem file source into ADLS. This is a file migration step.

### B. API ingestion
The pipeline API_Ingestion.json calls an API, gets JSON data, and copies it into ADLS. This is a good example of integrating external data sources into Azure.

### C. SQL to Data Lake
The pipeline SQLtoDatalake.json is the most important one for your interview because it implements the incremental load logic.

### D. Silver layer transformation
The pipeline SilverLayer.json runs the mapping data flow DataTransformation.json, which:
- cleans and standardizes columns,
- derives new values,
- filters records,
- writes data into Delta format in the silver layer.

### E. Gold layer serving
The pipeline GoldLayer.json runs DataServing.json, which joins the transformed datasets and creates business-oriented aggregate outputs like top-selling airlines and top-performing flights.

---

## How incremental load is implemented here

This is the part you should explain very clearly in the interview.

The incremental load is implemented in SQLtoDatalake.json, and it is a watermark-based incremental load pattern.

### What is meant by incremental load?
Instead of loading all historical rows every time, the pipeline loads only the new or updated rows since the last successful run.

That is better because:
- it saves time,
- reduces cost,
- makes the pipeline faster,
- avoids reprocessing old data.

---

## Deep explanation of the incremental load logic

The pipeline uses three main activities:

### 1. LastLoad
This is a Lookup activity that reads the previous watermark value from a JSON file stored in ADLS.

The idea is:
- “What was the last date we successfully loaded?”

That value is stored as a watermark.

### 2. LatestLoad
This is another Lookup activity that queries Azure SQL and finds the maximum booking date in the source table.

So it asks:
- “What is the latest date available in the source right now?”

### 3. CopySQLData
This is the actual data movement activity.

It uses a SQL query like this conceptually:

- fetch rows where booking_date is greater than the previous watermark
- and less than or equal to the latest available date

That means the pipeline only copies the new slice of data.

The important part of the query in the JSON is:

- previous watermark: from the LastLoad activity
- current max date: from the LatestLoad activity

So the pipeline is effectively doing:

- “Load only data that arrived after the last successful execution.”

---

## Why this is called a watermark

A watermark is basically a marker that says:
- “We processed up to this point.”

In this project, the watermark is a date value based on booking_date.

That means:
- first run: it loads all rows from the start up to the latest date,
- next run: it loads only rows newer than the previous watermark.

So the pipeline is maintaining a moving boundary of what has already been processed.

---

## How the watermark is stored and updated

The project stores the watermark in a JSON file through the “Watermark” activity.

That activity:
- reads from a simple JSON source,
- adds a new column called lastload,
- writes it back to the lookup dataset in ADLS.

So after the copy is successful, the pipeline overwrites the watermark file with the latest load date.

That means the next pipeline run will use that new value as the starting point.

This is the core of the incremental logic.

---

## In interview language, you can explain it like this

“I implemented incremental loading using a watermark pattern. Instead of reloading the entire table every time, I used a lookup activity to read the last successful load date from a JSON file in ADLS, and another lookup to get the maximum booking date from Azure SQL. Then I used those values in the source query to pull only rows where booking_date was greater than the previous watermark and less than or equal to the latest available date. After the load completed, I updated the watermark file with the new maximum date so the next run would pick up only the new data.”

That is exactly the right explanation.

---

## Why this is a good implementation

This approach is practical because:
- it is easy to understand,
- it is low-cost,
- it avoids full table reloads,
- it works well for date-based incremental loads.

It is a standard pattern used in many real-world Azure Data Factory projects.

---

## Important caveat

This implementation is good, but it is still a simplified version. It assumes:
- the source data is being appended over time,
- booking_date is a reliable indicator of new data,
- there are no major data corrections or late-arriving records.

In a production-grade system, a more robust approach could be:
- CDC from the source database,
- a control table in SQL,
- last modified timestamp instead of a business date,
- merge/upsert logic with audit columns.

So if you want to sound very strong in an interview, you can say:
- “This project uses a manual watermark-based incremental pattern, which is suitable for the current scope, but in enterprise-grade pipelines I would prefer CDC or a last-modified timestamp approach for higher reliability.”

---

## The big picture of the project

If I had to describe the whole project in one sentence, I would say:

“This is an Azure-based data engineering project that ingests data from on-prem, API, and Azure SQL sources; lands it in ADLS; transforms it through a medallion-style silver and gold layer architecture; and uses a watermark-driven incremental load pattern to process only new data efficiently.”

---

## Short version you can say in an interview

- “This project is built on Azure Data Factory.”
- “It orchestrates ingestion from multiple sources into ADLS.”
- “It uses Self-Hosted Integration Runtime for on-prem files.”
- “It applies transformations through mapping data flows.”
- “It uses Delta format in silver and gold layers.”
- “For incremental loading, it uses a watermark pattern with a last-load date stored in a JSON file.”
- “The next run compares the previous watermark against the maximum source date and loads only new rows.”

If you want, I can also turn this into a polished 2-minute interview answer you can memorize and speak aloud.