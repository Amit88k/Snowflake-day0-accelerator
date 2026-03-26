---
name: day0-pipelines
description: Use when setting up data ingestion and pipeline
  infrastructure for a DAY0 environment. Covers internal and
  external stages, file formats, COPY INTO patterns, Snowpipe
  auto-ingest, Tasks, Dynamic Tables, and Streams for
  change data capture and incremental processing.
tools:
  - snowflake_sql_execute
  - snowflake_object_search
---

# When to Use

- Customer needs stages created for data landing and ingestion
- Defining file formats for CSV, JSON, Parquet, Avro, ORC, XML
- Setting up COPY INTO for batch data loading
- Configuring Snowpipe for continuous/auto-ingest pipelines
- Building incremental pipelines using Streams and Tasks
- Setting up Dynamic Tables for declarative transformations
- Establishing change data capture (CDC) patterns
- Migrating from external tools to native Snowflake pipelines

# What This Skill Provides

Complete, production-ready data pipeline infrastructure for a
Snowflake DAY0 environment. Covers every layer of the ingestion
stack — storage, format, load, and orchestration — following
Snowflake best practices. All output is fully qualified,
parameterized SQL ready for human review before execution.

---

# Instructions

## Core Guardrails — Non-Negotiable, Always Apply

- ALWAYS use fully qualified names: DATABASE.SCHEMA.OBJECT
- ALWAYS use IF NOT EXISTS on every CREATE statement
- NEVER store credentials inline in stage definitions —
  always use STORAGE INTEGRATION objects for external stages
- NEVER use COPY INTO without specifying ON_ERROR behavior
- NEVER leave FILE_FORMAT undefined in COPY INTO — always
  reference a named file format object
- ALWAYS enable server-side encryption on internal stages
- NEVER grant OWNERSHIP on a stage to a non-admin role
- ALWAYS use separate stages per data source or domain —
  never mix unrelated data sources in a single stage
- ALWAYS specify PURGE = FALSE in COPY INTO for auditing —
  only set TRUE after confirming with customer
- NEVER use FORCE = TRUE in COPY INTO without explicit
  customer approval — it reloads already-loaded files
- ALWAYS add COMMENT on stages and pipes describing source,
  owner, and environment
- ALWAYS use a dedicated service role for Snowpipe —
  never use ACCOUNTADMIN or SYSADMIN for pipe execution
- NEVER create a pipe without first verifying the target
  table schema matches the source file format exactly

---

## Pipeline Architecture Decision Guide

Before generating any pipeline SQL, determine the correct pattern:

```
What is the ingestion pattern?
│
├── Scheduled batch load (hourly/daily)?
│   └── Stage + Named File Format + COPY INTO + Task
│
├── Continuous file arrival (S3/GCS/Azure event)?
│   └── External Stage + Snowpipe (auto-ingest)
│
├── Real-time CDC from source system?
│   └── Stream on source table + Task + MERGE
│
├── Declarative transformation on loaded data?
│   └── Dynamic Table
│
├── Internal file staging (Snowflake-managed)?
│   └── Internal Named Stage
│
└── Cross-database or third-party tool integration?
    └── External Stage + Storage Integration
```

---

## Layer 1 — Storage Integrations

Storage integrations are required for ALL external stage access.
Never embed cloud credentials directly in stage definitions.

### Storage Integration Guardrails

- ALWAYS create a storage integration before creating external stages
- TYPE = EXTERNAL_STAGE for all cloud storage integrations
- STORAGE_ALLOWED_LOCATIONS must be scoped to exact paths —
  never use wildcard '*' in production
- ENABLED = TRUE always
- After creation, provide the IAM role ARN or service account
  to the customer for trust policy configuration on their
  cloud provider side — document this as a manual step

### Pattern — AWS S3 Storage Integration

```sql
CREATE STORAGE INTEGRATION IF NOT EXISTS <ENV>_S3_INTEGRATION
  TYPE                      = EXTERNAL_STAGE
  STORAGE_PROVIDER          = 'S3'
  ENABLED                   = TRUE
  STORAGE_ALLOWED_LOCATIONS = ('s3://<BUCKET>/<PREFIX>/')
  COMMENT = 'S3 storage integration for <SOURCE> | <ENV> | Owner: <TEAM>';

-- Run after creation to get IAM values for cloud trust policy
DESC INTEGRATION <ENV>_S3_INTEGRATION;
```

### Pattern — Azure Blob Storage Integration

```sql
CREATE STORAGE INTEGRATION IF NOT EXISTS <ENV>_AZURE_INTEGRATION
  TYPE                      = EXTERNAL_STAGE
  STORAGE_PROVIDER          = 'AZURE'
  ENABLED                   = TRUE
  AZURE_TENANT_ID           = '<TENANT_ID>'
  STORAGE_ALLOWED_LOCATIONS = ('azure://<ACCOUNT>.blob.core.windows.net/<CONTAINER>/<PREFIX>/')
  COMMENT = 'Azure storage integration for <SOURCE> | <ENV> | Owner: <TEAM>';
```

### Pattern — GCS Storage Integration

```sql
CREATE STORAGE INTEGRATION IF NOT EXISTS <ENV>_GCS_INTEGRATION
  TYPE                      = EXTERNAL_STAGE
  STORAGE_PROVIDER          = 'GCS'
  ENABLED                   = TRUE
  STORAGE_ALLOWED_LOCATIONS = ('gcs://<BUCKET>/<PREFIX>/')
  COMMENT = 'GCS storage integration for <SOURCE> | <ENV> | Owner: <TEAM>';
```

---

## Layer 2 — Named File Formats

Always create named file format objects — never define inline format options directly in COPY INTO or stage definitions. Named formats are reusable, versioned, and auditable.

### File Format Guardrails

- ALWAYS create a named FILE FORMAT object — never use inline options
- One file format per distinct file structure — do not reuse formats across incompatible sources
- For CSV: ALWAYS define FIELD_OPTIONALLY_ENCLOSED_BY and NULL_IF to handle nulls and quoted fields correctly
- For JSON: set STRIP_OUTER_ARRAY = TRUE when loading arrays
- For Parquet: set USE_LOGICAL_TYPE = TRUE to preserve types
- ALWAYS set SKIP_HEADER = 1 for CSV files with headers
- ALWAYS set DATE_FORMAT and TIMESTAMP_FORMAT explicitly for CSV — never rely on auto-detection in production
- For semi-structured: ALWAYS set COMPRESSION = AUTO unless format is confirmed uncompressed

### Pattern — CSV File Format (Standard)

```sql
CREATE FILE FORMAT IF NOT EXISTS <DB>.<SCHEMA>.<ENV>_CSV_FORMAT
  TYPE                      = 'CSV'
  FIELD_DELIMITER           = ','
  RECORD_DELIMITER          = '\n'
  SKIP_HEADER               = 1
  FIELD_OPTIONALLY_ENCLOSED_BY = '"'
  NULL_IF                   = ('NULL', 'null', '', 'N/A')
  EMPTY_FIELD_AS_NULL       = TRUE
  DATE_FORMAT               = 'YYYY-MM-DD'
  TIMESTAMP_FORMAT          = 'YYYY-MM-DD HH24:MI:SS'
  TRIM_SPACE                = TRUE
  ERROR_ON_COLUMN_COUNT_MISMATCH = TRUE
  COMMENT = 'Standard CSV format | <ENV> | Owner: <TEAM>';
```

### Pattern — JSON File Format

```sql
CREATE FILE FORMAT IF NOT EXISTS <DB>.<SCHEMA>.<ENV>_JSON_FORMAT
  TYPE             = 'JSON'
  COMPRESSION      = 'AUTO'
  STRIP_OUTER_ARRAY = TRUE
  STRIP_NULL_VALUES = FALSE
  IGNORE_UTF8_ERRORS = FALSE
  COMMENT = 'Standard JSON format | <ENV> | Owner: <TEAM>';
```

### Pattern — Parquet File Format

```sql
CREATE FILE FORMAT IF NOT EXISTS <DB>.<SCHEMA>.<ENV>_PARQUET_FORMAT
  TYPE             = 'PARQUET'
  COMPRESSION      = 'AUTO'
  USE_LOGICAL_TYPE = TRUE
  COMMENT = 'Standard Parquet format | <ENV> | Owner: <TEAM>';
```

### Pattern — Avro File Format

```sql
CREATE FILE FORMAT IF NOT EXISTS <DB>.<SCHEMA>.<ENV>_AVRO_FORMAT
  TYPE        = 'AVRO'
  COMPRESSION = 'AUTO'
  COMMENT = 'Standard Avro format | <ENV> | Owner: <TEAM>';
```

### Pattern — ORC File Format

```sql
CREATE FILE FORMAT IF NOT EXISTS <DB>.<SCHEMA>.<ENV>_ORC_FORMAT
  TYPE        = 'ORC'
  TRIM_SPACE  = TRUE
  COMMENT = 'Standard ORC format | <ENV> | Owner: <TEAM>';
```

---

## Layer 3 — Stages

### Internal Named Stage (Snowflake-Managed)

Use when data is uploaded directly to Snowflake without cloud storage intermediary.

### Internal Stage Guardrails

- ALWAYS enable server-side encryption on internal stages
- Use DIRECTORY = (ENABLE = TRUE) when stage will be queried via directory table functions
- NEVER use the user stage (@~) or table stage for production pipelines — always use named internal stages

### Pattern — Internal Named Stage

```sql
CREATE STAGE IF NOT EXISTS <DB>.<SCHEMA>.<ENV>_<SOURCE>_STAGE
  ENCRYPTION     = (TYPE = 'SNOWFLAKE_SSE')
  DIRECTORY      = (ENABLE = TRUE)
  COMMENT = 'Internal stage for <SOURCE> data | <ENV> | Owner: <TEAM>';
```

### External Stage — AWS S3

```sql
CREATE STAGE IF NOT EXISTS <DB>.<SCHEMA>.<ENV>_<SOURCE>_S3_STAGE
  URL               = 's3://<BUCKET>/<PREFIX>/'
  STORAGE_INTEGRATION = <ENV>_S3_INTEGRATION
  FILE_FORMAT       = <DB>.<SCHEMA>.<ENV>_<FORMAT>_FORMAT
  DIRECTORY         = (ENABLE = TRUE)
  COMMENT = 'S3 external stage for <SOURCE> | <ENV> | Owner: <TEAM>';
```

### External Stage — Azure Blob

```sql
CREATE STAGE IF NOT EXISTS <DB>.<SCHEMA>.<ENV>_<SOURCE>_AZURE_STAGE
  URL                 = 'azure://<ACCOUNT>.blob.core.windows.net/<CONTAINER>/<PREFIX>/'
  STORAGE_INTEGRATION = <ENV>_AZURE_INTEGRATION
  FILE_FORMAT         = <DB>.<SCHEMA>.<ENV>_<FORMAT>_FORMAT
  DIRECTORY           = (ENABLE = TRUE)
  COMMENT = 'Azure external stage for <SOURCE> | <ENV> | Owner: <TEAM>';
```

### External Stage — GCS

```sql
CREATE STAGE IF NOT EXISTS <DB>.<SCHEMA>.<ENV>_<SOURCE>_GCS_STAGE
  URL               = 'gcs://<BUCKET>/<PREFIX>/'
  STORAGE_INTEGRATION = <ENV>_GCS_INTEGRATION
  FILE_FORMAT       = <DB>.<SCHEMA>.<ENV>_<FORMAT>_FORMAT
  DIRECTORY         = (ENABLE = TRUE)
  COMMENT = 'GCS external stage for <SOURCE> | <ENV> | Owner: <TEAM>';
```

---

## Layer 4 — COPY INTO (Batch Loading)

### COPY INTO Guardrails

- ALWAYS reference a named FILE FORMAT — never use inline options
- ALWAYS specify ON_ERROR — never leave default (ABORT_STATEMENT) without confirming with customer
- ALWAYS set PURGE = FALSE unless customer explicitly confirms they want files deleted post-load
- NEVER use FORCE = TRUE without customer written approval — it bypasses load history and causes duplicate data
- For large files: use PATTERN to load by date partition to avoid loading entire stage at once
- ALWAYS use MATCH_BY_COLUMN_NAME for semi-structured formats to avoid column order dependencies
- After every COPY INTO, run a validation query to confirm row counts match expected source counts
- Use ON_ERROR = 'CONTINUE' only when partial loads are acceptable — document this decision in comments

### ON_ERROR Decision Guide

```
What should happen when a row fails to load?
        │
        ├── Abort entire load (safest for critical data)?
        │       └── ON_ERROR = 'ABORT_STATEMENT'
        │
        ├── Skip bad rows, continue loading rest?
        │       └── ON_ERROR = 'CONTINUE'
        │
        ├── Skip bad rows up to a limit, then abort?
        │       └── ON_ERROR = 'SKIP_FILE_<N>' or
        │           ON_ERROR = 'SKIP_FILE_<N>%'
        │
        └── Not sure? Use ABORT_STATEMENT and confirm with customer.
```

### Pattern — Standard COPY INTO (CSV)

```sql
COPY INTO <DB>.<SCHEMA>.<TARGET_TABLE>
FROM   @<DB>.<SCHEMA>.<ENV>_<SOURCE>_STAGE
FILE_FORMAT = (FORMAT_NAME = '<DB>.<SCHEMA>.<ENV>_CSV_FORMAT')
PATTERN     = '.*<SOURCE>_.*[.]csv'
ON_ERROR    = 'ABORT_STATEMENT'
PURGE       = FALSE;
```

### Pattern — COPY INTO with Column Mapping (CSV)

```sql
COPY INTO <DB>.<SCHEMA>.<TARGET_TABLE>
  (COL1, COL2, COL3, LOAD_TIMESTAMP)
FROM (
  SELECT
    $1,
    $2,
    $3,
    CURRENT_TIMESTAMP()
  FROM @<DB>.<SCHEMA>.<ENV>_<SOURCE>_STAGE
)
FILE_FORMAT = (FORMAT_NAME = '<DB>.<SCHEMA>.<ENV>_CSV_FORMAT')
ON_ERROR    = 'ABORT_STATEMENT'
PURGE       = FALSE;
```

### Pattern — COPY INTO from JSON (Semi-Structured)

```sql
COPY INTO <DB>.<SCHEMA>.<TARGET_TABLE>
FROM (
  SELECT
    $1:id::VARCHAR          AS ID,
    $1:name::VARCHAR        AS NAME,
    $1:created_at::TIMESTAMP AS CREATED_AT,
    $1                      AS RAW_JSON,
    CURRENT_TIMESTAMP()     AS LOAD_TIMESTAMP,
    METADATA$FILENAME       AS SOURCE_FILE,
    METADATA$FILE_ROW_NUMBER AS FILE_ROW
  FROM @<DB>.<SCHEMA>.<ENV>_<SOURCE>_STAGE
)
FILE_FORMAT = (FORMAT_NAME = '<DB>.<SCHEMA>.<ENV>_JSON_FORMAT')
ON_ERROR    = 'ABORT_STATEMENT'
PURGE       = FALSE;
```

### Pattern — COPY INTO from Parquet

```sql
COPY INTO <DB>.<SCHEMA>.<TARGET_TABLE>
FROM (
  SELECT
    $1:col1::VARCHAR   AS COL1,
    $1:col2::NUMBER    AS COL2,
    $1:col3::DATE      AS COL3,
    METADATA$FILENAME  AS SOURCE_FILE,
    CURRENT_TIMESTAMP() AS LOAD_TIMESTAMP
  FROM @<DB>.<SCHEMA>.<ENV>_<SOURCE>_STAGE
)
FILE_FORMAT         = (FORMAT_NAME = '<DB>.<SCHEMA>.<ENV>_PARQUET_FORMAT')
MATCH_BY_COLUMN_NAME = CASE_INSENSITIVE
ON_ERROR            = 'ABORT_STATEMENT'
PURGE               = FALSE;
```

### Pattern — Validation After COPY INTO

Always append this after every COPY INTO:

```sql
-- Confirm row count after load
SELECT COUNT(*) AS LOADED_ROWS
FROM   <DB>.<SCHEMA>.<TARGET_TABLE>
WHERE  LOAD_TIMESTAMP >= DATEADD('minute', -10, CURRENT_TIMESTAMP());

-- Review any load errors
SELECT *
FROM   TABLE(INFORMATION_SCHEMA.COPY_HISTORY(
         TABLE_NAME    => '<TARGET_TABLE>',
         START_TIME    => DATEADD('hour', -1, CURRENT_TIMESTAMP())
       ))
ORDER  BY LAST_LOAD_TIME DESC;
```

---

## Layer 5 — Snowpipe (Continuous Auto-Ingest)

### When to Use Snowpipe

Use Snowpipe when files arrive continuously to a cloud stage and need to be loaded within minutes without a scheduled batch. Snowpipe uses event notifications from S3/Azure/GCS to trigger loads automatically — no scheduler required.

### Snowpipe Guardrails

- ALWAYS use a dedicated service role for pipe ownership — never use ACCOUNTADMIN or SYSADMIN
- ALWAYS create a notification integration before creating the pipe for auto-ingest
- AUTO_INGEST = TRUE for cloud event-driven ingestion — requires SQS (AWS), Event Grid (Azure), or Pub/Sub (GCS) notification configured on the cloud provider side
- ALWAYS enable COPY_OPTIONS = (ON_ERROR = 'CONTINUE') for Snowpipe — failed rows should not block the pipe
- NEVER use FORCE = TRUE in pipe definitions
- Monitor pipe health via SYSTEM$PIPE_STATUS() after creation
- Snowpipe uses serverless compute — track costs via SNOWFLAKE.ACCOUNT_USAGE.PIPE_USAGE_HISTORY
- ALWAYS document the cloud event notification ARN/URL in the COMMENT field after setup

### Pattern — Notification Integration (AWS SQS)

```sql
CREATE NOTIFICATION INTEGRATION IF NOT EXISTS <ENV>_S3_NOTIFY_INTEGRATION
  TYPE           = QUEUE
  NOTIFICATION_PROVIDER = AWS_SQS
  ENABLED        = TRUE
  AWS_SQS_ARN    = 'arn:aws:sqs:<REGION>:<ACCOUNT_ID>:<QUEUE_NAME>'
  COMMENT = 'SQS notification integration for Snowpipe | <ENV> | Owner: <TEAM>';

-- Run after creation to get Snowflake SQS subscription details
DESC INTEGRATION <ENV>_S3_NOTIFY_INTEGRATION;
```

### Pattern — Notification Integration (Azure Event Grid)

```sql
CREATE NOTIFICATION INTEGRATION IF NOT EXISTS <ENV>_AZURE_NOTIFY_INTEGRATION
  TYPE                        = QUEUE
  NOTIFICATION_PROVIDER       = AZURE_STORAGE_QUEUE
  ENABLED                     = TRUE
  AZURE_STORAGE_QUEUE_PRIMARY_URI = 'https://<ACCOUNT>.queue.core.windows.net/<QUEUE>'
  AZURE_TENANT_ID             = '<TENANT_ID>'
  COMMENT = 'Azure Event Grid notification integration | <ENV> | Owner: <TEAM>';
```

### Pattern — Snowpipe with Auto-Ingest (AWS S3)

```sql
CREATE PIPE IF NOT EXISTS <DB>.<SCHEMA>.<ENV>_<SOURCE>_PIPE
  AUTO_INGEST   = TRUE
  COMMENT = 'Auto-ingest pipe for <SOURCE> from S3 | <ENV>
             | Owner: <TEAM> | SQS ARN: <ARN>'
AS
COPY INTO <DB>.<SCHEMA>.<TARGET_TABLE>
FROM   @<DB>.<SCHEMA>.<ENV>_<SOURCE>_S3_STAGE
FILE_FORMAT = (FORMAT_NAME = '<DB>.<SCHEMA>.<ENV>_<FORMAT>_FORMAT')
ON_ERROR    = 'CONTINUE';

-- Verify pipe status after creation
SELECT SYSTEM$PIPE_STATUS('<DB>.<SCHEMA>.<ENV>_<SOURCE>_PIPE');
```

### Pattern — Snowpipe with Column Mapping

```sql
CREATE PIPE IF NOT EXISTS <DB>.<SCHEMA>.<ENV>_<SOURCE>_PIPE
  AUTO_INGEST = TRUE
  COMMENT = 'Auto-ingest pipe for <SOURCE> | <ENV> | Owner: <TEAM>'
AS
COPY INTO <DB>.<SCHEMA>.<TARGET_TABLE>
  (COL1, COL2, COL3, LOAD_TIMESTAMP, SOURCE_FILE)
FROM (
  SELECT
    $1:col1::VARCHAR,
    $1:col2::NUMBER,
    $1:col3::DATE,
    CURRENT_TIMESTAMP(),
    METADATA$FILENAME
  FROM @<DB>.<SCHEMA>.<ENV>_<SOURCE>_S3_STAGE
)
FILE_FORMAT = (FORMAT_NAME = '<DB>.<SCHEMA>.<ENV>_JSON_FORMAT')
ON_ERROR    = 'CONTINUE';
```

### Snowpipe Role Grants (Dedicated Service Role)

```sql
-- Create dedicated pipe service role
CREATE ROLE IF NOT EXISTS <ENV>_PIPE_SVC_ROLE
  COMMENT = 'Service role for Snowpipe execution | <ENV>';

-- Grant minimum required privileges
GRANT USAGE  ON DATABASE <DB>              TO ROLE <ENV>_PIPE_SVC_ROLE;
GRANT USAGE  ON SCHEMA   <DB>.<SCHEMA>     TO ROLE <ENV>_PIPE_SVC_ROLE;
GRANT INSERT ON TABLE    <DB>.<SCHEMA>.<TABLE> TO ROLE <ENV>_PIPE_SVC_ROLE;
GRANT READ   ON STAGE    <DB>.<SCHEMA>.<STAGE> TO ROLE <ENV>_PIPE_SVC_ROLE;
GRANT OPERATE ON PIPE    <DB>.<SCHEMA>.<PIPE>  TO ROLE <ENV>_PIPE_SVC_ROLE;
```

---

## Layer 6 — Streams (Change Data Capture)

### When to Use Streams

Use Streams to capture INSERT, UPDATE, and DELETE changes on a source table for incremental processing. Streams are the foundation of CDC pipelines in Snowflake.

### Stream Guardrails

- ALWAYS use APPEND_ONLY = TRUE for insert-only source tables — reduces overhead and simplifies downstream processing
- Default stream mode captures all DML — use for full CDC
- ALWAYS consume the stream inside a transaction to ensure the offset advances atomically
- NEVER read a stream in a SELECT without processing it in the same transaction — uncommitted reads do not advance offset
- Monitor stream staleness via STALE column in SHOW STREAMS — streams become stale after 14 days of no consumption
- ALWAYS create streams on the LANDING or RAW layer table, not on transformed tables

### Pattern — Standard CDC Stream

```sql
CREATE STREAM IF NOT EXISTS <DB>.<SCHEMA>.<ENV>_<SOURCE>_STREAM
  ON TABLE <DB>.<SCHEMA>.<SOURCE_TABLE>
  COMMENT = 'CDC stream on <SOURCE_TABLE> | <ENV> | Owner: <TEAM>';
```

### Pattern — Append-Only Stream (Insert-Only Source)

```sql
CREATE STREAM IF NOT EXISTS <DB>.<SCHEMA>.<ENV>_<SOURCE>_STREAM
  ON TABLE  <DB>.<SCHEMA>.<SOURCE_TABLE>
  APPEND_ONLY = TRUE
  COMMENT = 'Append-only stream on <SOURCE_TABLE> | <ENV> | Owner: <TEAM>';
```

---

## Layer 7 — Tasks (Pipeline Orchestration)

### When to Use Tasks

Use Tasks to schedule SQL or stored procedure execution on a defined cadence. Tasks are the native Snowflake scheduler for pipeline orchestration — use instead of external schedulers where possible.

### Task Guardrails

- ALWAYS use serverless tasks (no warehouse specified) for simple transformation tasks — more cost-efficient
- Only assign a warehouse to a task when the workload requires specific warehouse sizing (e.g., Snowpark Python tasks)
- ALWAYS set a SCHEDULE — never leave a task without one
- Use AFTER clause to chain dependent tasks — never use external orchestration for intra-Snowflake dependencies
- ALWAYS include error handling in the task SQL or procedure
- NEVER start a task root without testing it manually first using EXECUTE TASK
- ALWAYS resume tasks explicitly after creation — they start in SUSPENDED state by default
- Monitor via SNOWFLAKE.ACCOUNT_USAGE.TASK_HISTORY

### Pattern — Serverless Task (Simple SQL)

```sql
CREATE TASK IF NOT EXISTS <DB>.<SCHEMA>.<ENV>_<NAME>_TASK
  SCHEDULE    = 'USING CRON 0 2 * * * UTC'
  COMMENT = 'Scheduled pipeline task for <PURPOSE> | <ENV> | Owner: <TEAM>'
AS
INSERT INTO <DB>.<SCHEMA>.<TARGET_TABLE>
SELECT * FROM <DB>.<SCHEMA>.<SOURCE_TABLE>
WHERE  LOAD_TIMESTAMP >= DATEADD('hour', -1, CURRENT_TIMESTAMP());

-- Resume task after creation
ALTER TASK <DB>.<SCHEMA>.<ENV>_<NAME>_TASK RESUME;
```

### Pattern — Warehouse Task (Snowpark or Heavy Transform)

```sql
CREATE TASK IF NOT EXISTS <DB>.<SCHEMA>.<ENV>_<NAME>_TASK
  WAREHOUSE   = <ENV>_ETL_WH
  SCHEDULE    = 'USING CRON 0 1 * * * UTC'
  COMMENT = 'ETL transformation task | <ENV> | Owner: <TEAM>'
AS
CALL <DB>.<SCHEMA>.<TRANSFORM_PROCEDURE>();

ALTER TASK <DB>.<SCHEMA>.<ENV>_<NAME>_TASK RESUME;
```

### Pattern — Stream-Triggered Task (CDC Pipeline)

```sql
-- Root task: checks stream has data before proceeding
CREATE TASK IF NOT EXISTS <DB>.<SCHEMA>.<ENV>_<SOURCE>_ROOT_TASK
  SCHEDULE      = '5 MINUTE'
  WHEN SYSTEM$STREAM_HAS_DATA('<DB>.<SCHEMA>.<ENV>_<SOURCE>_STREAM')
  COMMENT = 'Stream-triggered CDC root task | <ENV> | Owner: <TEAM>'
AS
MERGE INTO <DB>.<SCHEMA>.<TARGET_TABLE> TGT
USING (
  SELECT * FROM <DB>.<SCHEMA>.<ENV>_<SOURCE>_STREAM
) SRC
ON TGT.ID = SRC.ID
WHEN MATCHED AND SRC.METADATA$ACTION = 'DELETE'
  THEN DELETE
WHEN MATCHED AND SRC.METADATA$ACTION = 'INSERT'
  THEN UPDATE SET
    TGT.COL1 = SRC.COL1,
    TGT.COL2 = SRC.COL2,
    TGT.UPDATED_AT = CURRENT_TIMESTAMP()
WHEN NOT MATCHED AND SRC.METADATA$ACTION = 'INSERT'
  THEN INSERT (ID, COL1, COL2, CREATED_AT)
  VALUES (SRC.ID, SRC.COL1, SRC.COL2, CURRENT_TIMESTAMP());

ALTER TASK <DB>.<SCHEMA>.<ENV>_<SOURCE>_ROOT_TASK RESUME;
```

---

## Layer 8 — Dynamic Tables

### When to Use Dynamic Tables

Use Dynamic Tables for declarative, automated transformations where you define WHAT the result should look like and Snowflake manages HOW and WHEN to refresh. Preferred over Tasks + Views for transformation pipelines with freshness SLAs.

### Dynamic Table Guardrails

- ALWAYS define TARGET_LAG to control freshness — never use DOWNSTREAM unless the table feeds another Dynamic Table
- Use TARGET_LAG = 'X minutes' for near-real-time requirements
- Use TARGET_LAG = 'X hours' for batch-style freshness
- ALWAYS specify the WAREHOUSE for the refresh compute
- NEVER define DML inside a Dynamic Table — it is a SELECT only
- Dynamic Tables do not support: streams, external tables as base, or non-deterministic functions like CURRENT_TIMESTAMP() directly
- ALWAYS monitor via DYNAMIC_TABLE_REFRESH_HISTORY in ACCOUNT_USAGE
- Use INITIALIZE = 'ON_CREATE' to populate immediately on creation

### Pattern — Dynamic Table (Near Real-Time)

```sql
CREATE DYNAMIC TABLE IF NOT EXISTS <DB>.<SCHEMA>.<ENV>_<NAME>_DT
  TARGET_LAG  = '5 minutes'
  WAREHOUSE   = <ENV>_ETL_WH
  INITIALIZE  = 'ON_CREATE'
  COMMENT = 'Dynamic table for <PURPOSE> | <ENV> | Owner: <TEAM>'
AS
SELECT
  ID,
  NAME,
  AMOUNT,
  CATEGORY,
  EVENT_DATE
FROM <DB>.<SCHEMA>.<SOURCE_TABLE>
WHERE IS_ACTIVE = TRUE;
```

### Pattern — Dynamic Table (Daily Batch Freshness)

```sql
CREATE DYNAMIC TABLE IF NOT EXISTS <DB>.<SCHEMA>.<ENV>_<NAME>_DT
  TARGET_LAG  = '1 hour'
  WAREHOUSE   = <ENV>_ETL_WH
  INITIALIZE  = 'ON_CREATE'
  COMMENT = 'Daily aggregation dynamic table | <ENV> | Owner: <TEAM>'
AS
SELECT
  DATE_TRUNC('day', EVENT_TIMESTAMP) AS EVENT_DATE,
  CATEGORY,
  COUNT(*)                            AS EVENT_COUNT,
  SUM(AMOUNT)                         AS TOTAL_AMOUNT
FROM <DB>.<SCHEMA>.<SOURCE_TABLE>
GROUP BY 1, 2;
```

---

## Pipeline Role Grants

After creating all pipeline objects, always generate grants:

```sql
-- Stage access
GRANT READ  ON STAGE <DB>.<SCHEMA>.<STAGE> TO ROLE <ENV>_ETL_ROLE;
GRANT WRITE ON STAGE <DB>.<SCHEMA>.<STAGE> TO ROLE <ENV>_LOADER_ROLE;

-- Pipe access
GRANT MONITOR ON PIPE <DB>.<SCHEMA>.<PIPE> TO ROLE <ENV>_ETL_ROLE;
GRANT OPERATE ON PIPE <DB>.<SCHEMA>.<PIPE> TO ROLE <ENV>_PIPE_SVC_ROLE;

-- Stream access
GRANT SELECT ON TABLE <DB>.<SCHEMA>.<SOURCE_TABLE>
  TO ROLE <ENV>_ETL_ROLE;

-- Task access
GRANT MONITOR ON TASK <DB>.<SCHEMA>.<TASK> TO ROLE <ENV>_ETL_ROLE;
GRANT OPERATE ON TASK <DB>.<SCHEMA>.<TASK> TO ROLE <ENV>_ETL_ROLE;

-- Dynamic Table access
GRANT SELECT ON DYNAMIC TABLE <DB>.<SCHEMA>.<DT>
  TO ROLE <ENV>_ANALYST_ROLE;
```

---

## Deployment Order

Always generate and deploy pipeline objects in this exact sequence:

1. Storage integrations (cloud trust — manual step required)
2. Notification integrations (for Snowpipe auto-ingest)
3. Named file formats
4. Internal and external stages
5. Landing / raw target tables
6. COPY INTO scripts (batch load patterns)
7. Snowpipe definitions
8. Streams on source tables
9. Tasks (root tasks last, chained tasks first)
10. Dynamic Tables
11. RESUME all tasks
12. Role grants on all pipeline objects
13. Validation queries

---

## Validation Queries

Always append after full deployment:

```sql
-- Confirm stages exist
SHOW STAGES IN SCHEMA <DB>.<SCHEMA>;

-- Confirm file formats exist
SHOW FILE FORMATS IN SCHEMA <DB>.<SCHEMA>;

-- Confirm pipes and status
SELECT SYSTEM$PIPE_STATUS('<DB>.<SCHEMA>.<PIPE_NAME>');

-- Confirm stream is active and not stale
SHOW STREAMS IN SCHEMA <DB>.<SCHEMA>;

-- Confirm task schedule and status
SHOW TASKS IN SCHEMA <DB>.<SCHEMA>;

-- Confirm dynamic table freshness
SELECT NAME, TARGET_LAG, SCHEDULING_STATE, LAST_COMPLETED_REFRESH_TIME
FROM INFORMATION_SCHEMA.DYNAMIC_TABLES
WHERE TABLE_SCHEMA = '<SCHEMA>';

-- Review COPY INTO history for last load
SELECT *
FROM TABLE(INFORMATION_SCHEMA.COPY_HISTORY(
  TABLE_NAME => '<TARGET_TABLE>',
  START_TIME => DATEADD('hour', -1, CURRENT_TIMESTAMP())
))
ORDER BY LAST_LOAD_TIME DESC;
```

---

# Examples

## Example 1: Full Batch Ingestion Pipeline (CSV from S3)

**User:** `$day0-pipelines` Set up a batch ingestion pipeline for
PROD. Source: CSV files landing in S3 bucket every hour.
Target: PROD_DB.RAW.ORDERS table. Owner: DATA_TEAM.
Load schedule: every hour at minute 5.

**Expected output:** S3 storage integration, external stage,
CSV file format, COPY INTO script, scheduled task,
validation queries, role grants.

## Example 2: Continuous Snowpipe from Azure Blob

**User:** `$day0-pipelines` Set up Snowpipe for continuous JSON
file ingestion from Azure Blob Storage into
PROD_DB.RAW.EVENTS. Owner: PLATFORM_TEAM. Files arrive
within seconds of being written.

**Expected output:** Azure storage integration, notification
integration, external stage, JSON file format, pipe with
AUTO_INGEST = TRUE, service role, role grants,
pipe status validation.

## Example 3: CDC Pipeline with Stream and Task

**User:** `$day0-pipelines` Build a CDC pipeline from
PROD_DB.RAW.CUSTOMERS (source) into
PROD_DB.CLEAN.CUSTOMERS (target). Run every 5 minutes.
Owner: DATA_ENGINEERING_TEAM.

**Expected output:** Append-only stream on source table,
stream-triggered task with MERGE statement, task
resumed, validation queries.

## Example 4: Dynamic Table Transformation Layer

**User:** `$day0-pipelines` Create a dynamic table for daily
revenue aggregation in PROD. Source: PROD_DB.RAW.ORDERS.
Target: PROD_DB.ANALYTICS.DAILY_REVENUE. Freshness: 1 hour.
Warehouse: PROD_ETL_WH. Owner: ANALYTICS_TEAM.

**Expected output:** Dynamic table with 1-hour TARGET_LAG,
SELECT aggregation, INITIALIZE = ON_CREATE, analyst
role grant, freshness validation query.
