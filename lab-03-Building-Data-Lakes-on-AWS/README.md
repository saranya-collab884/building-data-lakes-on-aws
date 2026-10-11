# Lab 03: Working with Data as a Product

Part of the **Building Data Lakes on AWS** hands-on series.

## Objective

AnyCompany is building an application that uses movie data and needs to serve two kinds of customers: **standard** and **enterprise** subscribers. In this lab I treated the movies dataset as a curated data product: I validated an AWS Glue job that keeps the processed data up to date, then used **Lake Formation tags (LF-tags)** to control who can see which databases, tables, and columns. Finally I signed in as two test consumers and confirmed each one sees exactly what their subscription allows.

**What I built:**
- Reviewed and validated a scheduled Glue job (`transform-movies`) that maintains the processed dataset
- Defined three LF-tags: `Environment`, `Customer`, and `Confidential`
- Applied the tags to a database, a table, and individual columns
- Granted tag-based permissions (LF-TBAC) to two consumers
- Verified each consumer's view of the data in Athena

## Architecture

<details>
<summary>Architecture diagram</summary>

![Lab 3 architecture](/screenshots/lab3/architecture.png)

</details>

## Services Used

| Service | Role in this lab |
|---|---|
| Amazon S3 | Stores the raw movies data and the processed output |
| AWS Glue ETL job and trigger | Cleans the data and appends new rows on a 10-minute schedule |
| AWS Glue Data Catalog | Holds the `transform-movies-db` database and `movies` table |
| AWS Lake Formation | Defines LF-tags and grants tag-based permissions |
| Amazon Athena | Used to validate the data and test each consumer's view |
| AWS IAM | Users `Consumer_A` and `Consumer_B` used to test access |

## Dataset

The movies dataset from Lab 1, processed by the `transform-movies` job into a table with **13 columns** and **4,609 rows**. The job adds a `rating_filled` column that fills in missing ratings.

> The raw dataset is not included in this repo.

## Steps Performed

### 1. Tested and validated the Glue job
Opened the `transform-movies` job and reviewed how it works. It is a visual job of eight nodes: it reads the raw CSV and the catalog table, fills missing ratings, maps the schema, appends any new rows with a custom PySpark step, and writes the result back to the catalog. A trigger runs it every 10 minutes, so I waited for a run to show **Succeeded**.

<details>
<summary>transform-movies job details</summary>

![Glue job details](/screenshots/lab3/glue-job-nodes.png)

</details>

<details>
<summary>Successful job run</summary>

![Glue job run succeeded](/screenshots/lab3/glue-job-succeeded.png)

</details>

Then validated the output in Athena as the data engineer:

```sql
SELECT * FROM "transform-movies-db"."movies" LIMIT 10;

SELECT COUNT(*) AS number_of_movies FROM "movies";
```

**Result:** 13 columns visible and 4,609 movies.

<details>
<summary>Engineer view: sample rows and row count</summary>

![Engineer sample](/screenshots/lab3/athena-engineer-sample.png)
![Engineer count](/screenshots/lab3/athena-engineer-count.png)

</details>

### 2. Defined the LF-tags
Created three tags in Lake Formation:

| Key | Values |
|---|---|
| `Environment` | Development, Production |
| `Customer` | Regular, Enterprise |
| `Confidential` | True, False |

<details>
<summary>LF-tags created</summary>

![LF-tags](/screenshots/lab3/lf-tags.png)

</details>

### 3. Applied the tags to resources
- **Database** `transform-movies-db`: `Environment = Production`
- **Table** `movies`: `Confidential = False` (it inherits `Environment = Production` from the database)
- **Columns:** `Customer = Regular` on the 11 general columns, and `Customer = Enterprise` on `rank` and `rating_filled`

<details>
<summary>Database tag</summary>

![Database LF-tag](/screenshots/lab3/tag-database.png)

</details>

<details>
<summary>Table tags (inherited and assigned)</summary>

![Table LF-tags](/screenshots/lab3/tag-table.png)

</details>

<details>
<summary>Column tags</summary>

![Column LF-tags](/screenshots/lab3/tag-columns.png)

</details>

### 4. Created the tag-based permissions
First I revoked the `IAMAllowedPrincipals` permissions on the database and table, so access is controlled by the tags rather than by default IAM access. Then I granted each consumer permissions based on tag values:

| Key | Data Engineer | Consumer_A | Consumer_B |
|---|---|---|---|
| Environment | Development, Production | Production | Production |
| Customer | Regular, Enterprise | Regular | Regular, Enterprise |
| Confidential | True, False | False | False |

Each consumer got two grants: `Describe` on databases tagged `Environment = Production`, and `Select` + `Describe` on tables matching their `Confidential` and `Customer` tags.

<details>
<summary>Consumer_A grants</summary>

![Consumer_A permissions](/screenshots/lab3/grant-consumer-a.png)

</details>

<details>
<summary>Consumer_B grants</summary>

![Consumer_B permissions](/screenshots/lab3/grant-consumer-b.png)

</details>

### 5. Verified each consumer's view
Signed in as each test user and ran the same query in Athena:

```sql
SELECT * FROM "transform-movies-db"."movies" LIMIT 10;
```

| User | Customer type | Columns visible |
|---|---|---|
| Consumer_A | Regular | 11 (no `rank` or `rating_filled`) |
| Consumer_B | Enterprise | 13 (all columns) |

<details>
<summary>Consumer_A view (11 columns)</summary>

![Consumer_A results](/screenshots/lab3/athena-consumer-a.png)

</details>

<details>
<summary>Consumer_B view (13 columns)</summary>

![Consumer_B results](/screenshots/lab3/athena-consumer-b.png)

</details>

## Results

| Item | Outcome |
|---|---|
| Glue job run status | Succeeded |
| Rows in `movies` | 4,609 |
| Columns for engineer | 13 |
| Columns for Consumer_A (regular) | 11 |
| Columns for Consumer_B (enterprise) | 13 |
| LF-tags defined | 3 (`Environment`, `Customer`, `Confidential`) |

## Key Learnings

- **Tags scale better than per-resource grants.** I granted access once per tag combination, and any resource with matching tags is covered, including new ones.
- **Tags are inherited.** The table picked up `Environment = Production` from its database, so only the differences need tagging.
- **Multiple tag keys in one grant combine with AND.** Consumer_A only gets tables where `Confidential = False` AND `Customer = Regular`.
- **Multiple values for one key combine with OR.** Consumer_B's `Customer = Regular, Enterprise` grant matches either value.
- **Column-level security comes from tagging columns.** The same table shows different columns to different users with no copies or views.
- **Revoking `IAMAllowedPrincipals` is what makes tags effective.** Without it, default IAM-based access would bypass the tag rules.
- **Treating data as a product means validating it first.** I checked the pipeline output before exposing it to consumers.


