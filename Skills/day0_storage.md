---
name: day0-storage
description: Use when setting up Snowflake storage infrastructure
  for a DAY0 environment. Covers database and schema design,
  table architecture patterns, data retention, clustering,
  search optimization, and data governance guardrails including
  tags, masking policies, and row access policies.
tools:
  - snowflake_sql_execute
  - snowflake_object_search
---

# When to Use

- Customer needs databases and schemas created for a new environment
- Designing table structures for raw, clean, and analytics layers
- Defining data retention and time travel policies
- Setting up column-level security via masking policies
- Configuring row-level security via row access policies
- Creating tag taxonomies for data classification
- Applying search optimization for selective query patterns
- Setting up table clustering for large partition-pruning benefit
- Establishing naming conventions across all storage objects

# What This Skill Provides

Complete, production-ready storage layer setup for a Snowflake
DAY0 environment. Covers database architecture, schema layering,
table design best practices, and governance controls. All output
is fully qualified, parameterized SQL ready for human review
before execution.

---

# Instructions

## Core Guardrails — Non-Negotiable, Always Apply

- ALWAYS use fully qualified names: DATABASE.SCHEMA.OBJECT
- ALWAYS use IF NOT EXISTS on every CREATE statement
- NEVER create a table without defining a primary identifier
  column — at minimum a surrogate key or natural key
- NEVER use FLOAT or DOUBLE for monetary or financial columns —
  always use NUMBER(precision, scale)
- NEVER use VARCHAR without a length in critical key columns —
  always define VARCHAR(n) for known-length fields
- ALWAYS add a COMMENT on every database, schema, and table
  describing purpose, owner, and environment
- NEVER drop or alter existing objects without explicit customer
  approval — always generate new objects and propose migration
- ALWAYS define DATA_RETENTION_TIME_IN_DAYS explicitly —
  never rely on account defaults
- NEVER use the PUBLIC schema for production data objects —
  always create named schemas with clear purpose
- ALWAYS separate raw, clean, and analytics layers into
  distinct schemas or databases
- NEVER mix PII and non-PII data in the same table without
  masking policies applied to PII columns
- ALWAYS apply tags to tables containing sensitive data
  before the table is populated
- NEVER grant SELECT on a table containing PII directly to
  analyst roles — always route through a masked view or
  apply a masking policy first

---

## Database and Schema Architecture

### Naming Convention Standards

Always enforce this naming pattern across all environments:

- Databases: `__DB` — e.g. PROD_RAW_DB, PROD_CLEAN_DB, PROD_ANALYTICS_DB
- Schemas: or `<SOURCE_SYSTEM>` — e.g. SALES, FINANCE, CUSTOMERS, MARKETING
- Tables: (singular noun, snake_case) — e.g. ORDER, CUSTOMER, PRODUCT, DAILY_REVENUE
- Stages: `__STAGE`
- Views: `_V` or `MASKED_V`
- Pipes: `PIPE`
- Tasks: `_TASK`

Never use abbreviations that are not universally understood.
Always use uppercase for all Snowflake object names.

### Three-Layer Architecture

Always generate databases following this layered model:

**Layer 1 — RAW (Landing)**
- **Purpose:** Exact copy of source data, no transformation
- **Schema:** One schema per source system
- **Tables:** LANDING tables only, append-only preferred
- **Access:** ETL roles write, analysts never access directly

**Layer 2 — CLEAN (Conformed)**
- **Purpose:** Typed, deduplicated, validated, conformed data
- **Schema:** One schema per business domain
- **Tables:** Dimension and fact tables, SCD2 where needed
- **Access:** BI tools, analysts with masking policies applied

**Layer 3 — ANALYTICS (Presentation)**
- **Purpose:** Aggregated, business-ready datasets and metrics
- **Schema:** One schema per use case or reporting domain
- **Tables:** Pre-aggregated tables, wide tables, feature stores
- **Access:** Broad analyst and reporting tool access

### Pattern — Full Three-Layer Database Setup

```sql
-- Layer 1: Raw
CREATE DATABASE IF NOT EXISTS <ENV>_RAW_DB
  DATA_RETENTION_TIME_IN_DAYS = 7
  COMMENT = 'Raw landing layer | <ENV> | Owner: <DATA_ENG_TEAM>';

CREATE SCHEMA IF NOT EXISTS <ENV>_RAW_DB.<SOURCE_SYSTEM>
  DATA_RETENTION_TIME_IN_DAYS = 7
  COMMENT = 'Raw data from <SOURCE_SYSTEM> | <ENV> | Owner: <DATA_ENG_TEAM>';

-- Layer 2: Clean
CREATE DATABASE IF NOT EXISTS <ENV>_CLEAN_DB
  DATA_RETENTION_TIME_IN_DAYS = 14
  COMMENT = 'Conformed and validated data | <ENV> | Owner: <DATA_ENG_TEAM>';

CREATE SCHEMA IF NOT EXISTS <ENV>_CLEAN_DB.<DOMAIN>
  DATA_RETENTION_TIME_IN_DAYS = 14
  COMMENT = '<DOMAIN> conformed schema | <ENV> | Owner: <DATA_ENG_TEAM>';

-- Layer 3: Analytics
CREATE DATABASE IF NOT EXISTS <ENV>_ANALYTICS_DB
  DATA_RETENTION_TIME_IN_DAYS = 30
  COMMENT = 'Analytics and reporting layer | <ENV> | Owner: <ANALYTICS_TEAM>';

CREATE SCHEMA IF NOT EXISTS <ENV>_ANALYTICS_DB.<DOMAIN>
  DATA_RETENTION_TIME_IN_DAYS = 30
  COMMENT = '<DOMAIN> analytics schema | <ENV> | Owner: <ANALYTICS_TEAM>';
```

---

## Data Retention and Time Travel

### Retention Policy Rules

- ALWAYS set DATA_RETENTION_TIME_IN_DAYS explicitly at both database and schema level — never rely on account default
- Raw layer: 7 days minimum (short retention, high volume)
- Clean layer: 14 days (medium retention for debugging)
- Analytics layer: 30 days (longer for business validation)
- Critical dimension tables: 90 days maximum (Enterprise)
- Dev and test environments: 1 day to reduce storage cost
- NEVER set retention to 0 on production objects — it disables time travel entirely which removes recovery capability
- ALWAYS set TRANSIENT = TRUE for staging or temp tables to avoid unnecessary time travel storage cost on ephemeral data

### Transient vs Permanent Tables

Use TRANSIENT tables when:
- Table is a staging/intermediate step in a pipeline
- Data is fully reproducible from source
- Time travel and Fail-safe are not needed
- Goal is to reduce storage cost on temporary data

Use PERMANENT tables when:
- Table is a source of truth
- Data cannot be easily reproduced
- Audit, recovery, or time travel is required
- Table is accessed by downstream applications

### Pattern — Transient Staging Table

```sql
CREATE TRANSIENT TABLE IF NOT EXISTS <DB>.<SCHEMA>.<ENV>_<NAME>_STG
  DATA_RETENTION_TIME_IN_DAYS = 0
  COMMENT = 'Transient staging table | <ENV> | Owner: <TEAM>'
AS SELECT * FROM <SOURCE_TABLE> WHERE 1 = 0;
```

---

## Table Design Patterns

### Column Design Guardrails

- ALWAYS include a surrogate key (NUMBER AUTOINCREMENT or VARCHAR UUID) as the first column on every dimension table
- ALWAYS include these audit columns on every table:
  - `CREATED_AT TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP()`
  - `UPDATED_AT TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP()`
  - `CREATED_BY VARCHAR(100) DEFAULT CURRENT_USER()`
  - `SOURCE_FILE VARCHAR(500)` for load traceability
- NEVER use TIMESTAMP_LTZ for warehouse data — use TIMESTAMP_NTZ and store timezone separately if needed
- ALWAYS use NUMBER(38,2) for currency/monetary amounts
- ALWAYS use VARCHAR(16777216) for free-text or unknown length fields — do not truncate user content
- NEVER use BOOLEAN where NULL is a valid state — use VARCHAR('Y','N','UNKNOWN') instead
- ALWAYS add NOT NULL constraint to primary key columns
- For JSON/semi-structured: store raw in VARIANT column AND extract typed columns — never VARIANT-only in clean layer

### Pattern — Standard Dimension Table

```sql
CREATE TABLE IF NOT EXISTS <DB>.<SCHEMA>.<ENV>_<ENTITY>
(
  <ENTITY>_SK       NUMBER        NOT NULL AUTOINCREMENT
                                  COMMENT 'Surrogate key',
  <ENTITY>_NK       VARCHAR(100)  NOT NULL
                                  COMMENT 'Natural/business key',
  NAME              VARCHAR(500),
  DESCRIPTION       VARCHAR(16777216),
  IS_ACTIVE         BOOLEAN       DEFAULT TRUE,
  EFFECTIVE_FROM    DATE,
  EFFECTIVE_TO      DATE,
  CREATED_AT        TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP(),
  UPDATED_AT        TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP(),
  CREATED_BY        VARCHAR(100)  DEFAULT CURRENT_USER(),
  SOURCE_SYSTEM     VARCHAR(200),
  CONSTRAINT PK_<ENTITY> PRIMARY KEY (<ENTITY>_SK)
)
DATA_RETENTION_TIME_IN_DAYS = 14
COMMENT = '<ENTITY> dimension table | <ENV> | Owner: <TEAM>';
```

### Pattern — Standard Fact Table

```sql
CREATE TABLE IF NOT EXISTS <DB>.<SCHEMA>.<ENV>_<FACT_NAME>_FACT
(
  <FACT>_SK         NUMBER        NOT NULL AUTOINCREMENT
                                  COMMENT 'Surrogate key',
  <DIM1>_SK         NUMBER        NOT NULL
                                  COMMENT 'FK to <DIM1>',
  <DIM2>_SK         NUMBER        NOT NULL
                                  COMMENT 'FK to <DIM2>',
  EVENT_DATE        DATE          NOT NULL,
  EVENT_TIMESTAMP   TIMESTAMP_NTZ NOT NULL,
  AMOUNT            NUMBER(38,2),
  QUANTITY          NUMBER(18,4),
  METRIC_1          NUMBER(38,6),
  CREATED_AT        TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP(),
  LOAD_TIMESTAMP    TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP(),
  SOURCE_FILE       VARCHAR(500),
  CONSTRAINT PK_<FACT> PRIMARY KEY (<FACT>_SK)
)
CLUSTER BY (EVENT_DATE, <DIM1>_SK)
DATA_RETENTION_TIME_IN_DAYS = 30
COMMENT = '<FACT> fact table | <ENV> | Owner: <TEAM>';
```

### Pattern — Raw Landing Table (Semi-Structured)

```sql
CREATE TABLE IF NOT EXISTS <DB>.<SCHEMA>.<ENV>_<SOURCE>_LANDING
(
  RECORD_ID         NUMBER        NOT NULL AUTOINCREMENT,
  RAW_DATA          VARIANT       NOT NULL
                                  COMMENT 'Raw source JSON/XML payload',
  SOURCE_FILE       VARCHAR(500)  NOT NULL,
  FILE_ROW_NUMBER   NUMBER,
  LOAD_TIMESTAMP    TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP(),
  BATCH_ID          VARCHAR(100),
  CONSTRAINT PK_<SOURCE>_LANDING PRIMARY KEY (RECORD_ID)
)
DATA_RETENTION_TIME_IN_DAYS = 7
COMMENT = 'Raw landing for <SOURCE> | <ENV> | Owner: <DATA_ENG_TEAM>';
```

### Pattern — SCD Type 2 Table

```sql
CREATE TABLE IF NOT EXISTS <DB>.<SCHEMA>.<ENV>_<ENTITY>_HISTORY
(
  <ENTITY>_SK       NUMBER        NOT NULL AUTOINCREMENT,
  <ENTITY>_NK       VARCHAR(100)  NOT NULL,
  NAME              VARCHAR(500),
  ATTRIBUTE_1       VARCHAR(500),
  ATTRIBUTE_2       NUMBER(38,2),
  EFFECTIVE_FROM    DATE          NOT NULL,
  EFFECTIVE_TO      DATE,
  IS_CURRENT        BOOLEAN       DEFAULT TRUE,
  HASH_KEY          VARCHAR(64)   COMMENT 'MD5 hash of tracked columns',
  CREATED_AT        TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP(),
  SOURCE_SYSTEM     VARCHAR(200),
  CONSTRAINT PK_<ENTITY>_HIST PRIMARY KEY (<ENTITY>_SK)
)
DATA_RETENTION_TIME_IN_DAYS = 90
COMMENT = 'SCD2 history for <ENTITY> | <ENV> | Owner: <TEAM>';
```

---

## Clustering

### When to Apply Clustering

- Table exceeds 1TB AND queries filter on specific columns consistently — never cluster small or medium tables
- Clustering is beneficial for date/timestamp columns on large fact tables with time-range filters
- NEVER cluster on high-cardinality columns like UUID or row-level timestamps — choose columns with low-medium cardinality that appear in WHERE clauses regularly
- Monitor clustering depth via SYSTEM$CLUSTERING_INFORMATION before and after applying — confirm benefit exists
- Automatic clustering incurs serverless compute cost — always set a budget monitor for clustering spend

### Clustering Guardrails

- Maximum 3–4 clustering keys per table
- Always put the lowest-cardinality key first (e.g., DATE before CATEGORY before ID)
- NEVER cluster transient or staging tables
- Prefer natural partition columns (DATE, REGION, STATUS) over synthetic keys

### Pattern — Add Clustering to Existing Table

```sql
ALTER TABLE <DB>.<SCHEMA>.<TABLE>
  CLUSTER BY (EVENT_DATE, REGION_CODE);

-- Verify clustering benefit
SELECT SYSTEM$CLUSTERING_INFORMATION(
  '<DB>.<SCHEMA>.<TABLE>',
  '(EVENT_DATE, REGION_CODE)'
);
```

---

## Search Optimization

### When to Apply Search Optimization

Use for tables where users run highly selective point-lookup queries (returning < 5% of rows) on non-clustered columns. Particularly useful for user-facing search, support tooling, and customer lookup patterns.

### Search Optimization Guardrails

- NEVER apply to tables queried with full table scans — it will not help and adds cost
- ALWAYS confirm selective query pattern exists before enabling
- Search optimization has a storage cost — document and budget for it explicitly
- Works best on: VARCHAR equality, NUMBER range, VARIANT paths

### Pattern — Enable Search Optimization

```sql
ALTER TABLE <DB>.<SCHEMA>.<TABLE>
  ADD SEARCH OPTIMIZATION ON EQUALITY(<COL1>, <COL2>);

-- Verify optimization is building
SHOW TABLES LIKE '<TABLE>' IN SCHEMA <DB>.<SCHEMA>;
```

---

## Data Governance — Tag Taxonomy

### Tag Guardrails

- ALWAYS create all tags in a dedicated governance schema — never scatter tags across business schemas
- Use a consistent tag value vocabulary — define allowed values at tag creation time using ALLOWED_VALUES
- ALWAYS apply tags BEFORE tables are populated with data
- Tags must be applied at the column level for PII data — table-level tags are supplementary only
- NEVER create duplicate tags with similar names — maintain a tag registry in the SKILL_REGISTRY table

### Pattern — Governance Schema and Tag Setup

```sql
-- Create dedicated governance schema
CREATE DATABASE  IF NOT EXISTS <ENV>_GOVERNANCE_DB
  COMMENT = 'Governance objects | <ENV> | Owner: <DATA_GOVERNANCE_TEAM>';

CREATE SCHEMA IF NOT EXISTS <ENV>_GOVERNANCE_DB.TAGS
  COMMENT = 'Tag taxonomy | <ENV> | Owner: <DATA_GOVERNANCE_TEAM>';

-- PII sensitivity tag
CREATE TAG IF NOT EXISTS <ENV>_GOVERNANCE_DB.TAGS.PII_SENSITIVITY
  ALLOWED_VALUES = ('PUBLIC', 'INTERNAL', 'CONFIDENTIAL',
                    'RESTRICTED', 'PII', 'SENSITIVE_PII')
  COMMENT = 'Data sensitivity classification tag | <ENV>';

-- Data domain tag
CREATE TAG IF NOT EXISTS <ENV>_GOVERNANCE_DB.TAGS.DATA_DOMAIN
  ALLOWED_VALUES = ('CUSTOMER', 'FINANCE', 'PRODUCT',
                    'OPERATIONS', 'HR', 'MARKETING', 'LEGAL')
  COMMENT = 'Business domain classification tag | <ENV>';

-- Data owner tag
CREATE TAG IF NOT EXISTS <ENV>_GOVERNANCE_DB.TAGS.DATA_OWNER
  COMMENT = 'Team or person responsible for this data | <ENV>';

-- Retention policy tag
CREATE TAG IF NOT EXISTS <ENV>_GOVERNANCE_DB.TAGS.RETENTION_POLICY
  ALLOWED_VALUES = ('7_DAYS', '30_DAYS', '90_DAYS',
                    '1_YEAR', '7_YEARS', 'INDEFINITE')
  COMMENT = 'Data retention classification tag | <ENV>';
```

### Pattern — Apply Tags to Tables and Columns

```sql
-- Apply table-level tags
ALTER TABLE <DB>.<SCHEMA>.<TABLE>
  SET TAG <ENV>_GOVERNANCE_DB.TAGS.DATA_DOMAIN    = 'CUSTOMER',
          <ENV>_GOVERNANCE_DB.TAGS.DATA_OWNER     = '<TEAM>',
          <ENV>_GOVERNANCE_DB.TAGS.RETENTION_POLICY = '90_DAYS';

-- Apply column-level PII tags
ALTER TABLE <DB>.<SCHEMA>.<TABLE>
  MODIFY COLUMN EMAIL
  SET TAG <ENV>_GOVERNANCE_DB.TAGS.PII_SENSITIVITY = 'PII';

ALTER TABLE <DB>.<SCHEMA>.<TABLE>
  MODIFY COLUMN DATE_OF_BIRTH
  SET TAG <ENV>_GOVERNANCE_DB.TAGS.PII_SENSITIVITY = 'SENSITIVE_PII';

ALTER TABLE <DB>.<SCHEMA>.<TABLE>
  MODIFY COLUMN FULL_NAME
  SET TAG <ENV>_GOVERNANCE_DB.TAGS.PII_SENSITIVITY = 'PII';
```

---

## Data Governance — Masking Policies

### Masking Policy Guardrails

- ALWAYS create masking policies in the governance schema — never in the same schema as the data table
- One policy per data type and sensitivity level combination — do not create redundant duplicate policies
- ALWAYS use USING (val) to reference the column value
- NEVER return NULL as the masked value for required columns — use a recognizable placeholder ('MASKED', 'XXXX')
- ALWAYS test masking policy with both privileged and unprivileged roles before applying to production tables
- Apply policies at the column level via ALTER TABLE MODIFY COLUMN — never inline in table DDL
- A column can have only ONE masking policy — replacing requires UNSET first

### Pattern — Email Masking Policy

```sql
CREATE MASKING POLICY IF NOT EXISTS
  <ENV>_GOVERNANCE_DB.TAGS.<ENV>_EMAIL_MASK
  AS (VAL VARCHAR) RETURNS VARCHAR ->
  CASE
    WHEN IS_ROLE_IN_SESSION('<ENV>_PII_ADMIN_ROLE') THEN VAL
    WHEN IS_ROLE_IN_SESSION('<ENV>_ANALYST_ROLE')
      THEN REGEXP_REPLACE(VAL, '.+\@', '****@')
    ELSE '***MASKED***'
  END
  COMMENT = 'Email masking policy | <ENV>';
```

### Pattern — Date of Birth Masking Policy

```sql
CREATE MASKING POLICY IF NOT EXISTS
  <ENV>_GOVERNANCE_DB.TAGS.<ENV>_DOB_MASK
  AS (VAL DATE) RETURNS DATE ->
  CASE
    WHEN IS_ROLE_IN_SESSION('<ENV>_PII_ADMIN_ROLE') THEN VAL
    ELSE DATE_FROM_PARTS(YEAR(VAL), 1, 1)
  END
  COMMENT = 'DOB masking — returns year only for non-PII roles | <ENV>';
```

### Pattern — Phone Number Masking Policy

```sql
CREATE MASKING POLICY IF NOT EXISTS
  <ENV>_GOVERNANCE_DB.TAGS.<ENV>_PHONE_MASK
  AS (VAL VARCHAR) RETURNS VARCHAR ->
  CASE
    WHEN IS_ROLE_IN_SESSION('<ENV>_PII_ADMIN_ROLE') THEN VAL
    ELSE CONCAT('XXX-XXX-', RIGHT(VAL, 4))
  END
  COMMENT = 'Phone masking — shows last 4 digits only | <ENV>';
```

### Pattern — Generic Sensitive String Masking

```sql
CREATE MASKING POLICY IF NOT EXISTS
  <ENV>_GOVERNANCE_DB.TAGS.<ENV>_STRING_MASK
  AS (VAL VARCHAR) RETURNS VARCHAR ->
  CASE
    WHEN IS_ROLE_IN_SESSION('<ENV>_PII_ADMIN_ROLE') THEN VAL
    ELSE '***RESTRICTED***'
  END
  COMMENT = 'Generic string masking policy | <ENV>';
```

### Pattern — Apply Masking Policy to Column

```sql
ALTER TABLE <DB>.<SCHEMA>.<TABLE>
  MODIFY COLUMN EMAIL
  SET MASKING POLICY <ENV>_GOVERNANCE_DB.TAGS.<ENV>_EMAIL_MASK;

ALTER TABLE <DB>.<SCHEMA>.<TABLE>
  MODIFY COLUMN DATE_OF_BIRTH
  SET MASKING POLICY <ENV>_GOVERNANCE_DB.TAGS.<ENV>_DOB_MASK;

ALTER TABLE <DB>.<SCHEMA>.<TABLE>
  MODIFY COLUMN PHONE_NUMBER
  SET MASKING POLICY <ENV>_GOVERNANCE_DB.TAGS.<ENV>_PHONE_MASK;
```

---

## Data Governance — Row Access Policies

### Row Access Policy Guardrails

- Use row access policies to restrict which ROWS a role can see — distinct from masking which restricts COLUMN VALUES
- ALWAYS store the access mapping in a dedicated mapping table — never hardcode role names in the policy expression
- ALWAYS test with at least two roles (privileged and restricted) before applying to production
- A table can have only ONE row access policy — plan carefully
- Row access policies add query overhead — test performance impact on large tables before applying

### Pattern — Row Access Mapping Table

```sql
CREATE TABLE IF NOT EXISTS
  <ENV>_GOVERNANCE_DB.TAGS.<ENV>_ROW_ACCESS_MAP
(
  ROLE_NAME   VARCHAR(200) NOT NULL,
  REGION_CODE VARCHAR(50)  NOT NULL,
  CONSTRAINT PK_ROW_MAP PRIMARY KEY (ROLE_NAME, REGION_CODE)
)
DATA_RETENTION_TIME_IN_DAYS = 14
COMMENT = 'Row access mapping table | <ENV> | Owner: <DATA_GOVERNANCE_TEAM>';

-- Populate with initial mappings
INSERT INTO <ENV>_GOVERNANCE_DB.TAGS.<ENV>_ROW_ACCESS_MAP
  VALUES
  ('<ENV>_APAC_ANALYST_ROLE', 'APAC'),
  ('<ENV>_EMEA_ANALYST_ROLE', 'EMEA'),
  ('<ENV>_AMER_ANALYST_ROLE', 'AMER'),
  ('<ENV>_GLOBAL_ANALYST_ROLE', 'APAC'),
  ('<ENV>_GLOBAL_ANALYST_ROLE', 'EMEA'),
  ('<ENV>_GLOBAL_ANALYST_ROLE', 'AMER');
```

### Pattern — Row Access Policy (Region-Based)

```sql
CREATE ROW ACCESS POLICY IF NOT EXISTS
  <ENV>_GOVERNANCE_DB.TAGS.<ENV>_REGION_ROW_POLICY
  AS (REGION_CODE VARCHAR) RETURNS BOOLEAN ->
  IS_ROLE_IN_SESSION('<ENV>_DATA_ADMIN_ROLE')
  OR EXISTS (
    SELECT 1
    FROM <ENV>_GOVERNANCE_DB.TAGS.<ENV>_ROW_ACCESS_MAP
    WHERE ROLE_NAME   = CURRENT_ROLE()
    AND   REGION_CODE = REGION_CODE
  )
  COMMENT = 'Region-based row access policy | <ENV>';
```

### Pattern — Apply Row Access Policy to Table

```sql
ALTER TABLE <DB>.<SCHEMA>.<TABLE>
  ADD ROW ACCESS POLICY
    <ENV>_GOVERNANCE_DB.TAGS.<ENV>_REGION_ROW_POLICY
  ON (REGION_CODE);
```

---

## Storage Role Grants

After creating all storage objects, always generate grants following the principle of least privilege:

```sql
-- Database and schema access
GRANT USAGE ON DATABASE <DB>               TO ROLE <ENV>_ANALYST_ROLE;
GRANT USAGE ON SCHEMA   <DB>.<SCHEMA>      TO ROLE <ENV>_ANALYST_ROLE;

-- Table access — read only for analysts
GRANT SELECT ON ALL TABLES IN SCHEMA <DB>.<SCHEMA>
  TO ROLE <ENV>_ANALYST_ROLE;
GRANT SELECT ON FUTURE TABLES IN SCHEMA <DB>.<SCHEMA>
  TO ROLE <ENV>_ANALYST_ROLE;

-- ETL role — write access to raw layer only
GRANT INSERT, UPDATE ON ALL TABLES IN SCHEMA <ENV>_RAW_DB.<SCHEMA>
  TO ROLE <ENV>_ETL_ROLE;
GRANT INSERT, UPDATE ON FUTURE TABLES IN SCHEMA <ENV>_RAW_DB.<SCHEMA>
  TO ROLE <ENV>_ETL_ROLE;

-- Governance policy management
GRANT APPLY MASKING POLICY ON ACCOUNT
  TO ROLE <ENV>_DATA_GOVERNANCE_ROLE;
GRANT APPLY ROW ACCESS POLICY ON ACCOUNT
  TO ROLE <ENV>_DATA_GOVERNANCE_ROLE;
GRANT APPLY TAG ON ACCOUNT
  TO ROLE <ENV>_DATA_GOVERNANCE_ROLE;
```

---

## Deployment Order

Always generate and deploy storage objects in this exact sequence:

1. Governance database and schema
2. Tag objects (taxonomy first, before any tables)
3. Masking policies (before PII tables are created)
4. Row access mapping tables
5. Row access policies
6. Raw layer database and schemas
7. Clean layer database and schemas
8. Analytics layer database and schemas
9. Landing/raw tables
10. Clean/conformed tables
11. Analytics/aggregated tables
12. Apply tags to tables and columns
13. Apply masking policies to PII columns
14. Apply row access policies to restricted tables
15. Clustering on large tables (if applicable)
16. Search optimization (if applicable)
17. Role grants on all objects
18. Validation queries

---

## Validation Queries

Always append after full deployment:

```sql
-- Confirm databases exist
SHOW DATABASES LIKE '<ENV>%';

-- Confirm schemas in each layer
SHOW SCHEMAS IN DATABASE <ENV>_RAW_DB;
SHOW SCHEMAS IN DATABASE <ENV>_CLEAN_DB;
SHOW SCHEMAS IN DATABASE <ENV>_ANALYTICS_DB;

-- Confirm tables and retention settings
SELECT
  TABLE_CATALOG,
  TABLE_SCHEMA,
  TABLE_NAME,
  TABLE_TYPE,
  ROW_COUNT,
  RETENTION_TIME,
  COMMENT
FROM INFORMATION_SCHEMA.TABLES
WHERE TABLE_CATALOG LIKE '<ENV>%'
ORDER BY TABLE_CATALOG, TABLE_SCHEMA, TABLE_NAME;

-- Confirm tags exist
SHOW TAGS IN SCHEMA <ENV>_GOVERNANCE_DB.TAGS;

-- Confirm masking policies exist
SHOW MASKING POLICIES IN SCHEMA <ENV>_GOVERNANCE_DB.TAGS;

-- Confirm masking policies applied to columns
SELECT *
FROM SNOWFLAKE.ACCOUNT_USAGE.POLICY_REFERENCES
WHERE POLICY_KIND = 'MASKING_POLICY'
AND REF_DATABASE_NAME LIKE '<ENV>%';

-- Confirm row access policies applied
SELECT *
FROM SNOWFLAKE.ACCOUNT_USAGE.POLICY_REFERENCES
WHERE POLICY_KIND = 'ROW_ACCESS_POLICY'
AND REF_DATABASE_NAME LIKE '<ENV>%';

-- Confirm tag assignments on PII columns
SELECT
  OBJECT_DATABASE,
  OBJECT_SCHEMA,
  OBJECT_NAME,
  COLUMN_NAME,
  TAG_NAME,
  TAG_VALUE
FROM SNOWFLAKE.ACCOUNT_USAGE.TAG_REFERENCES
WHERE TAG_NAME = 'PII_SENSITIVITY'
AND OBJECT_DATABASE LIKE '<ENV>%';
```

---

# Examples

## Example 1: Full Three-Layer Storage Stack

**User:** `$day0-storage` Set up full three-layer database
architecture for PROD. Domains: CUSTOMERS, ORDERS, FINANCE.
Owner: DATA_PLATFORM_TEAM. Retention: standard per layer.

**Expected output:** 3 databases (RAW, CLEAN, ANALYTICS),
3 schemas per database (one per domain), governance DB
with tag taxonomy, masking policies for email/phone/DOB,
role grants per layer, validation queries.

## Example 2: PII-Heavy Table Setup

**User:** `$day0-storage` Create a CUSTOMER dimension table in
PROD_CLEAN_DB.CUSTOMERS. Columns include: customer ID,
full name, email, phone, date of birth, region, account
status. Apply all PII governance controls.

**Expected output:** Dimension table DDL with audit columns,
PII tags on name/email/phone/DOB columns, masking policies
applied to all PII columns, row access policy on REGION,
analyst role grants with masking in place.

## Example 3: Raw Landing for JSON Source

**User:** `$day0-storage` Create a raw landing table for JSON
event data from a mobile app. Schema: DEV_RAW_DB.MOBILE.
Source fields: event_id, user_id, event_type, payload,
event_timestamp. Needs to support reprocessing.

**Expected output:** PERMANENT landing table with VARIANT
column for raw JSON, audit columns, 7-day retention,
ETL role grants, no masking (raw layer).

## Example 4: Analytics Layer with Clustering

**User:** `$day0-storage` Create a DAILY_REVENUE fact table in
PROD_ANALYTICS_DB.FINANCE. Expected size: 2TB+.
Queries always filter by EVENT_DATE and REGION_CODE.

**Expected output:** Fact table with NUMBER(38,2) for amounts,
CLUSTER BY (EVENT_DATE, REGION_CODE), 30-day retention,
SYSTEM$CLUSTERING_INFORMATION validation query,
analyst role grants.
