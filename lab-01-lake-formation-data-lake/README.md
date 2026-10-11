# Lab 01: Build a Data Lake Using AWS Lake Formation

Part of the **Building Data Lakes on AWS** hands-on series.

## Objective

AnyCompany, a cloud marketing organization, wanted to pilot a data lake. In this lab I took a raw movies dataset (CSV) stored in Amazon S3 and turned it into a governed, queryable, analytics-ready data lake.

**What I built:**
- Registered an S3 location with AWS Lake Formation
- Created a database in the AWS Glue Data Catalog
- Used a Glue crawler to automatically discover the schema
- Queried the data with Amazon Athena
- Converted the data from CSV to Parquet using an AWS Glue ETL job
- Compared query performance between CSV and Parquet

## Architecture

![Lab 1 Architecture](/building-data-lakes-on-aws/screenshots/lab1/environment-overview.png)

## Services Used

| Service | Role in this lab |
|---|---|
| Amazon S3 | Stores the raw CSV data, the Parquet output, and Athena query results |
| AWS Lake Formation | Registers the storage location and manages permissions |
| AWS Glue Data Catalog | Stores metadata (databases, tables, schemas) |
| AWS Glue Crawler | Scans S3 and creates table definitions automatically |
| AWS Glue ETL (Visual ETL) | Transforms CSV into Parquet |
| Amazon Athena | Runs serverless SQL queries on the data in S3 |
| AWS IAM | Service roles used by Lake Formation and Glue |

## Dataset

A single CSV file, `movies.csv`, containing movie data from 1920 to 2018.

**Columns include:** `year`, `title`, `directors_0`, `rating`, `genres_0`,`genres_1`, `rank`, `running_time_secs`, `actors_0`,`actors_1`,`actors_2`,`directors_1`,`directors_2`


## Steps Performed

### 1. Explored the environment
Reviewed the S3 bucket layout: a `data/` folder holding the dataset and a `results/` folder for Athena query output.

![S3 structure](/building-data-lakes-on-aws/screenshots/lab1/aws_s3_folders.png)
![S3 structure](/building-data-lakes-on-aws/screenshots/lab1/aws_s3_data.png)

### 2. Set up Lake Formation
- **Registered the S3 `data/` location** as a data lake location using a dedicated service role, so Lake Formation can manage access to it.
- **Granted data location permission** to the Glue service role so the crawler and job could use the location.
- **Created the `movies-db` database** in the Glue Data Catalog.

![Registered location](/building-data-lakes-on-aws/screenshots/lab1/lake-formation-location.png)
![Database](/building-data-lakes-on-aws/screenshots/lab1/aws_lakehouse_formation_db.png)

### 3. Crawled the data with AWS Glue
- Created a crawler (`movies-table`) pointing at the S3 data location, targeting `movies-db`.
- Ran it and waited for the state to return to **Ready**.
- Verified the generated table schema against the original CSV columns.

![Crawler config](/building-data-lakes-on-aws/screenshots/lab1/crawler-config.png)
![Crawler run](/building-data-lakes-on-aws/screenshots/lab1/crawler-ready.png)
![Table schema](/building-data-lakes-on-aws/screenshots/lab1/Table_schema.png)

### 4. Queried the data with Athena
Granted my user `SELECT` permission on the table through Lake Formation, then ran queries in Athena.

```sql
SELECT * FROM "data" LIMIT 10;

SELECT COUNT(*) AS action_movies FROM "data" WHERE genres_0 = 'Action';
```

**Result:** 1002 movies with `Action` as the primary genre, matching the expected value.

![Athena sample](/building-data-lakes-on-aws/screenshots/lab1/sample.png)
![Athena count](/building-data-lakes-on-aws/screenshots/lab1/count2.png)

### 5. Transformed CSV to Parquet with a Glue job
Built a Visual ETL job (`ParquetConversion`):
- **Source:** Glue Data Catalog, `movies-db` / `data`
- **Target:** Amazon S3, Parquet format, uncompressed
- **Role:** Glue service role



![Visual job](/building-data-lakes-on-aws/screenshots/lab1/parquet_conv.png)
![Job succeeded](/building-data-lakes-on-aws/screenshots/lab1/job_run.png)

### 6. Cataloged the Parquet output
Re-ran the crawler so it discovered the new Parquet folder. The catalog now contained three tables: `data`, `movies_csv`, and `movies_parquet`.

![Tables list](/building-data-lakes-on-aws/screenshots/lab1/tables_list.png)

### 7. Compared CSV and Parquet in Athena

```sql
SELECT COUNT(*) AS action_movies FROM "movies_csv"     WHERE genres_0 = 'Action';
SELECT COUNT(*) AS action_movies FROM "movies_parquet" WHERE genres_0 = 'Action';
```

## Results

| Metric | CSV | Parquet |
|---|---|---|
| Time in queue | 56 ms | 97 ms |
| Run time | 484 ms | 532 ms |
| Data scanned |486.27 KB| 6.24 KB |
| Result count | 1002 | 1002 |

![Performance comparison](/building-data-lakes-on-aws/screenshots/lab1/count3.png)
![Performance comparison](/building-data-lakes-on-aws/screenshots/lab1/count4.png)

**Takeaway:** Both queries returned the same answer, but Parquet scanned far less data. Athena charges by data scanned, so on large datasets this translates directly into lower cost and faster queries.

## Key Learnings

- **Lake Formation adds a governance layer on top of IAM.** Even with the right IAM permissions, I needed an explicit Lake Formation `SELECT` grant before Athena could query a table.
- **The Data Catalog stores metadata only.** Tables are definitions pointing at data in S3; they do not hold the data themselves.
- **Crawlers save manual work.** They detect format and schema and register tables automatically, and they can be re-run to pick up new data.
- **Columnar formats matter.** Parquet stores data by column, so queries that touch a few columns read only those columns, unlike row-based CSV.
- **Athena is serverless.** There is no database to load or manage; it queries files in S3 directly using the catalog.

## Challenges and How I Solved Them

| Challenge | Solution |
|---|---|
| Athena returned a permissions error on a table | Granted `SELECT` (and grantable) permission to my user in Lake Formation → Tables → Actions → Grant |
| Tables did not appear right away after a crawler run | Waited for the crawler state to return to Ready, then refreshed the table list |





