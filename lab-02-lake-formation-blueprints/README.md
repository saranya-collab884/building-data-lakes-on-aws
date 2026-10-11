# Lab 02: Automate Data Lake Creation Using AWS Lake Formation Blueprints

Part of the **Building Data Lakes on AWS** hands-on series.

## Objective

In Lab 1 I built a data lake step by step by hand. In this lab, AnyCompany's operational analytics team needed AWS CloudTrail logs in a data lake so they could query them with Amazon Athena. Instead of repeating the manual setup, I used a **Lake Formation blueprint** to generate and run the whole ingestion pipeline as an AWS Glue workflow.

**What I built:**
- Registered an S3 location with Lake Formation and created a `cloudtraillogs-db` database
- Created a Glue workflow from the CloudTrail blueprint (`lf-cloudtrail-workflow`)
- Ran and monitored the workflow until the CloudTrail logs were loaded as Parquet
- Built a smaller custom workflow (`sample-workflow`) to understand how the blueprint's workflow is put together
- Validated the result in Athena and queried the logs for errors

## Architecture

<details>
<summary>Architecture diagram</summary>

![Lab 2 architecture](/screenshots/lab2/architecture.png)

</details>

## Services Used

| Service | Role in this lab |
|---|---|
| AWS CloudTrail | Source of the activity logs delivered to S3 |
| Amazon S3 | Stores raw CloudTrail logs, Parquet output, and Athena query results |
| AWS Lake Formation | Registers the location, manages permissions, and hosts the blueprint |
| AWS Glue Workflows | Orchestrates the crawler, jobs, and triggers as one tracked unit |
| AWS Glue Crawler / ETL jobs | Discover the schema and convert the logs to Parquet |
| AWS Glue Data Catalog | Stores the database and table metadata |
| Amazon Athena | Runs SQL queries on the resulting tables |
| AWS IAM | Service roles used by Lake Formation and the workflow |

## Steps Performed

### 1. Explored the environment
Checked the S3 bucket (`data/` for the data lake, `results/` for Athena output) and reviewed the `LabCloudTrail` trail to confirm where its logs are delivered. This trail is the source for the blueprint.

<details>
<summary>S3 bucket structure</summary>

![S3 bucket structure](/screenshots/lab2/s3-bucket-structure.png)

</details>

<details>
<summary>CloudTrail trail</summary>

![CloudTrail trail](/screenshots/lab2/cloudtrail-trail.png)

</details>

### 2. Set up Lake Formation
- **Registered the S3 `data/` location** with a service role, using Lake Formation permission mode.
- **Granted data location access** to the workflow role so the blueprint's jobs can write to the location.
- **Created the `cloudtraillogs-db` database** in the Glue Data Catalog.

<details>
<summary>Registered data lake location</summary>

![Registered location](/screenshots/lab2/lf-location.png)

</details>

<details>
<summary>Data location grant for the workflow role</summary>

![Data location grant](/screenshots/lab2/data-location-grant.png)

</details>

<details>
<summary>cloudtraillogs-db database</summary>

![Database](/screenshots/lab2/cloudtraillogs-db.png)

</details>

### 3. Created a workflow from a Lake Formation blueprint
Used the **AWS CloudTrail** blueprint with these settings:

| Setting | Value |
|---|---|
| Source | `LabCloudTrail` |
| Target database | `cloudtraillogs-db` |
| Data format | Parquet |
| Frequency | Run on demand |
| Workflow name | `lf-cloudtrail-workflow` |
| Table prefix | `lab` |
| IAM role | Workflow role |

<details>
<summary>lf-cloudtrail-workflow</summary>

![Blueprint configuration](/screenshots/lab2/lf-cloudtrail-workflow.png)

</details>

<details>
<summary>Workflow created</summary>

![Workflow created](/screenshots/lab2/workflow-created.png)

</details>

### 4. Ran and monitored the workflow
Started the workflow and followed its run in the Glue console as a directed acyclic graph (DAG). The run took roughly 15-20 minutes. The main line of the graph moves through these stages:

1. A pre-crawler job sets the workflow state to discovering
2. A crawler discovers the schema of the raw logs
3. A post-crawler job sets the state to importing
4. An ETL job converts the data to Parquet
5. A post-ETL job marks the workflow completed

<details>
<summary>Workflow Details (DAG)</summary>

![Workflow running](/screenshots/lab2/workflow-details-dag.png)

</details>

<details>
<summary>Workflow DAG</summary>

![Workflow completed](/screenshots/lab2/workflow-dag.png)

</details>

### 5. Validated the results
The catalog contained two tables:

- `_lab_cloudtrail`: the raw data before conversion
- `lab_cloudtrail`: the converted Parquet data (20 columns)

<details>
<summary>Tables in the Data Catalog</summary>

![Tables list](/screenshots/lab2/tables-list.png)

</details>

<details>
<summary>lab_cloudtrail schema</summary>

![Table schema](/screenshots/lab2/lab-cloudtrail-schema.png)

</details>

### 6. Queried the data with Athena

```sql
SELECT * FROM "cloudtraillogs-db"."lab_cloudtrail" LIMIT 10;
```

<details>
<summary>Sample query results</summary>

![Athena sample](/screenshots/lab2/athena-sample.png)

</details>

**Challenge: finding all errors in the logs.** Records that have an error code can be selected like this:

```sql
SELECT * FROM "cloudtraillogs-db"."lab_cloudtrail" WHERE NOT errorcode = '';
```

<details>
<summary>Error query results</summary>

![Athena errors](/screenshots/lab2/athena-errors.png)

</details>

## Results

| Item | Outcome |
|---|---|
| Workflow run status | Completed |
| Tables created | `_lab_cloudtrail` (raw), `lab_cloudtrail` (Parquet) |
| Columns in `lab_cloudtrail` | 20 |
| Rows with an error code | 99 |
| Workflow run time | 06 min 25 s |

## Key Learnings

- **Blueprints turn a manual build into a repeatable one.** A few form fields replaced the crawler, jobs, and triggers I assembled by hand in Lab 1.
- **A Glue workflow is a DAG that is run and tracked as one unit.** Each node is a job, crawler, or trigger, so progress and failures are visible per node.
- **Triggers connect the steps.** Each stage starts only when the stage it watches has finished.
- **The blueprint keeps raw and converted data side by side.** The raw table is kept alongside the Parquet table, which helps with auditing and reprocessing.
- **Workflows can run on demand or on a schedule.** A scheduled run would keep the data lake updated with new CloudTrail logs automatically.
- **Lake Formation permissions still apply.** The workflow role needed an explicit data location grant to write to the storage location.


