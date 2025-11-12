# NYC Taxi 2019 — PySpark Data Processing Pipeline

## Overview
This repository contains a PySpark pipeline (Jupyter Notebook) that demonstrates distributed data processing, lazy evaluation, and optimization strategies on NYC Yellow Taxi trip data for 2019 (~1+ GB).

## Dataset
Source: NYC TLC Yellow Taxi trip records (2019).  
Files used: `yellow_tripdata_2019-01.parquet` ... `yellow_tripdata_2019-12.parquet` (combined).  
Column subset used: `tpep_pickup_datetime, tpep_dropoff_datetime, trip_distance, passenger_count, total_amount, tip_amount, PULocationID, DOLocationID`.

## Pipeline steps
1. Load all monthly Parquet files with a single wildcard.
2. Column pruning (select only needed columns).
3. Data cleaning and derived columns using `withColumn` (trip_seconds, cost_per_mile, year, month).
4. Filter early for year==2019, passenger_count in [1,6], trip_distance between (0,100).
5. Complex aggregation across months to calculate average tip rate per hour of day and pickup ID.
6. Aggregations: trips by borough, trips per day, trips by pickup hour.
7. Partitioned write to Parquet.
8. Two SQL queries executed through `spark.sql()`.

## Optimization strategies applied
- **Predicate pushdown & column pruning**: Selecting only needed columns before heavy ops to reduce I/O.
- **Filters early**: Filters are applied prior to aggregations to reduce data size.
- **Repartitioning** before repeated groupBy by the same key to minimize future shuffle costs.
- **Partitioned outputs**: saved by `pickup_date` to enable faster reads when querying specific dates.

## Performance analysis (summary)
- To test caching, I compared the time required to count the records in my processed DataFrame before and after attempting to persist it in memory. I think that Databricks Serverless does not support .cache() or .persist(), so Spark effectively re-executed the query both times, resulting in the second count taking longer than the first.
- Spark automatically applied optimizations: predicate pushdown, column pruning, tungsten execution, and whole-stage code generation.
- Filter and aggregation operations were efficient due to Parquet pushdown.
- Writing to DBFS Public Root failed because Serverless does not support /FileStore/tables access — all output was written to the Volumes path instead.
- Caching and persisting were unavailable, but Spark still benefited from lazy evaluation and query reuse.

## How to run
1. Place Parquet files in DBFS (or update `base_path` in the notebook).
2. Open `NYC_taxi_pipeline.ipynb` in Databricks or locally with a properly configured Spark session.
3. Run cells top-to-bottom.

## Key findings (example)
- Spark’s lazy evaluation ensures transformations are only executed when an action is triggered.
- Catalyst Optimizer and Tungsten Engine handle most optimizations automatically.
- Even without explicit caching, Databricks Serverless efficiently handles large-scale Parquet data.
- Distributed aggregation scales well across 12 months of taxi data.
