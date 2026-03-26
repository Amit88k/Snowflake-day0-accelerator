---
name: day0-allobjects
description: Complete Day 0 Snowflake environment builder combining
  infrastructure (databases, schemas), compute (warehouse v1, v2,
  Snowpark-optimized, serverless, resource monitors), and RBAC
  (functional roles, access roles, future grants, cross-schema
  grants, service accounts, cost-attribution tags, teardown scripts).
  Covers all object types in a single interactive deployment.
  Supports bulk RUN execution with batched approvals.
tools:
  - snowflake_sql_execute
  - snowflake_object_search
  - ask_user_question
---

# When to Use

- Full Day 0 environment setup (databases + compute + RBAC)
- Adding databases, schemas, or warehouses to an existing environment
- Choosing between warehouse v1, v2 (Intelligence), or Snowpark-optimized
- Designing RBAC with functional and access roles
- Configuring resource monitors and credit limits
- Setting up cost-attribution tags and future grants
- Creating service account users for automation
- Cross-schema read grants for ELT pipelines
- Generating teardown/rollback scripts

# What This Skill Provides

Single-skill deployment of ALL Snowflake account objects: databases,
schemas, warehouses (all types), resource monitors, RBAC hierarchy,
future grants, tags, service users, and audit queries. Follows
Snowflake best practices. Batched RUN execution (2-4 approvals).

# Instructions

## Interactive Discovery Protocol

### Phase 1: Environment & Naming

Ask the user for:
1. **Environment names**: DEV, STG, UAT, PRD, etc.
2. **Naming style**: ENV_NAME_DB or NAME_ENV_DB or ENV_NAME
3. **Owner team**: Who owns this infrastructure?

STOPPING POINT: Confirm environments and naming.

### Phase 2: Databases & Schemas

For each environment, ask:
1. **Databases**: Functional databases (RAW, ANALYTICS, STAGE, etc.)
2. **Schemas per database**: e.g., INGESTION, TRANSFORM, CURATED
3. **Schema type**: Managed access vs. regular per schema
4. **Data retention**: Time Travel days per environment
5. **Transient preference**: Which DBs/schemas should be transient?

STOPPING POINT: Present Database/Schema manifest.

### Phase 3: Compute

For each environment, ask:
1. **Workload types**: ETL, analyst, BI, Snowpark, ML, Cortex AI?
2. **Warehouse strategy**: One per team? One per workload? Shared?
3. **Warehouse types**: Standard v1, Intelligence v2, Snowpark-optimized?
4. **Sizing**: Accept defaults or specify per warehouse
5. **Resource monitors**: Per-warehouse monitors? (Recommended: YES)

STOPPING POINT: Present Warehouse manifest.

### Phase 4: RBAC

Ask the user for:
1. **Functional teams**: DATA_ENGINEER, ANALYST, DATA_SCIENTIST, etc.
2. **Access mapping**: Per team → DB/schema → privilege level
3. **Role naming**: Confirm AR and FR naming patterns
4. **Future grants**: Enable on all schemas? (Recommended: YES)
5. **Cross-schema reads**: e.g., TRANSFORM reads from INGESTION?
6. **Warehouse assignments**: Which warehouse per functional role?

STOPPING POINT: Present Role-to-Object mapping matrix.

### Phase 5: Optional Add-Ons

1. **Cost-attribution tags**: Environment, team, cost_center?
2. **Service account users**: For SVC roles?
3. **Teardown script**: Rollback script alongside deployment?
4. **Default warehouse assignment**: Per functional role?

Defaults if skipped: tags=YES, service users=if SVC exists,
teardown=YES, default warehouse=YES.

## Core Guardrails (Non-Negotiable)

### Infrastructure Guardrails
- ALWAYS use fully qualified names: DATABASE.SCHEMA.OBJECT
- ALWAYS use IF NOT EXISTS on every CREATE statement
- ALWAYS set DATA_RETENTION_TIME_IN_DAYS on every database
- ALWAYS add COMMENT describing purpose, owner, environment
- ALWAYS use WITH MANAGED ACCESS on production schemas
- NEVER use OR REPLACE on databases/schemas (destroys data)

### Compute Guardrails
- ALWAYS include AUTO_SUSPEND on every warehouse — NEVER omit
- ALWAYS include AUTO_RESUME = TRUE
- ALWAYS set STATEMENT_TIMEOUT_IN_SECONDS — never unbounded
- NEVER set AUTO_SUSPEND below 60 seconds
- NEVER use WAREHOUSE_SIZE = LARGE+ without explicit approval
- ALWAYS prefix warehouses with environment name
- ALWAYS assign a resource monitor to every warehouse
- NEVER grant MODIFY on warehouses to non-admin roles

### RBAC Guardrails
- ALWAYS use SYSADMIN to create databases, schemas, warehouses
- ALWAYS use USERADMIN to create roles
- ALWAYS use SECURITYADMIN for GRANT statements
- ALWAYS grant all custom roles up to SYSADMIN
- NEVER create objects with ACCOUNTADMIN
- NEVER grant privileges directly to users — always via roles
- NEVER grant ALL PRIVILEGES on any object
- ALWAYS include both ALL and FUTURE grants
- NEVER omit USAGE on parent DB/schema

## Warehouse Type Decision Guide

```
Customer workload described?
│
├── Predictable, scheduled batch / ETL?
│   └── Warehouse v1 (STANDARD)
│       Best for: data loading, transforms, admin tasks
│
├── Interactive, variable, analyst / BI?
│   └── Warehouse v1 multi-cluster or v2
│       v2 if using Cortex AI or Snowflake Intelligence
│
├── Cortex AI, Snowflake Intelligence, mixed AI/analyst?
│   └── Warehouse v2 (SNOWFLAKE_INTELLIGENCE)
│       Optimized for variable AI + SQL workloads
│
├── Python, Pandas, Snowpark DataFrame workloads?
│   └── Snowpark-optimized warehouse
│       16x memory per node vs standard
│
├── ML training, feature engineering, model inference?
│   └── Snowpark-optimized (MEDIUM+ justified)
│       Set STATEMENT_TIMEOUT = 14400 for training
│
└── Serverless tasks, pipes, search, clustering?
    └── No warehouse needed — serverless compute
        Monitor via SERVERLESS_TASK_HISTORY
```

## Database Type Decision Guide

```
What is the database purpose?
│
├── Long-lived production data with audit/recovery?
│   └── Permanent database (retention 7–90 days)
│
├── Staging or ephemeral processing?
│   └── Transient database (retention 0–1 day)
│
├── Sandbox / developer experimentation?
│   └── Transient database (retention 0)
│
└── Shared / marketplace consumer?
    └── Created FROM SHARE
```

## Schema Type Decision Guide

```
What is the schema purpose?
│
├── Production / governed data?
│   └── Permanent + WITH MANAGED ACCESS
│
├── Raw ingestion data?
│   └── Permanent (managed access if multiple writers)
│
├── Temporary / staging?
│   └── Transient schema (retention 0–1)
│
└── Developer sandbox?
    └── Transient (in transient database)
```

## Environment Template Defaults

| Setting              | DEV        | UAT/STG     | PRD          |
|----------------------|------------|-------------|--------------|
| Database type        | Transient  | Permanent   | Permanent    |
| Schema managed access| No         | Yes         | Yes          |
| Retention days       | 0–1        | 1–3         | 7–14         |
| Warehouse size       | XSMALL     | SMALL       | SMALL–MEDIUM |
| Auto-suspend (sec)   | 60         | 60          | 120          |
| Monitor quota        | 20 credits | 50 credits  | 150 credits  |
| Tags                 | Optional   | Recommended | Required     |

## Warehouse Sizing Reference

| Workload             | Type                  | Size       | Suspend | Timeout |
|----------------------|-----------------------|------------|---------|---------|
| Admin / Monitoring   | STANDARD              | XSMALL     | 60s     | 1800s   |
| Ad-hoc Analyst       | STANDARD              | XSMALL–SM  | 60s     | 1800s   |
| Reporting / BI       | STANDARD or v2        | SMALL      | 60s     | 3600s   |
| ETL / Data Loading   | STANDARD              | SMALL–MED  | 120s    | 7200s   |
| Heavy Transformation | STANDARD              | MEDIUM     | 180s    | 7200s   |
| Cortex AI / LLM      | SNOWFLAKE_INTELLIGENCE| SMALL      | 60s     | 3600s   |
| Snowpark Python      | SNOWPARK-OPTIMIZED    | MEDIUM     | 120s    | 3600s   |
| ML Training          | SNOWPARK-OPTIMIZED    | LARGE*     | 120s    | 14400s  |

*LARGE requires explicit customer approval.

## Credit Quota Reference

| Size   | Std Credits/hr | Snowpark Credits/hr | DEV Quota | PRD Quota |
|--------|----------------|---------------------|-----------|-----------|
| XSMALL | 1              | N/A                 | 20–50     | 50–150    |
| SMALL  | 2              | N/A                 | 40–100    | 100–300   |
| MEDIUM | 4              | 6                   | 80–200    | 200–500   |
| LARGE  | 8              | 12                  | 200–500   | 500+      |

## Warehouse Patterns

### Pattern: Standard ETL Warehouse (v1)

```sql
CREATE WAREHOUSE IF NOT EXISTS <ENV>_ETL_WH
  WAREHOUSE_SIZE               = 'SMALL'
  WAREHOUSE_TYPE               = 'STANDARD'
  AUTO_SUSPEND                 = 120
  AUTO_RESUME                  = TRUE
  MAX_CONCURRENCY_LEVEL        = 4
  STATEMENT_TIMEOUT_IN_SECONDS = 7200
  COMMENT = 'ETL and data loading | <ENV> | Owner: <TEAM>';
```

### Pattern: Multi-Cluster Analyst Warehouse (v1)

```sql
CREATE WAREHOUSE IF NOT EXISTS <ENV>_ANALYST_WH
  WAREHOUSE_SIZE               = 'SMALL'
  WAREHOUSE_TYPE               = 'STANDARD'
  AUTO_SUSPEND                 = 60
  AUTO_RESUME                  = TRUE
  MIN_CLUSTER_COUNT            = 1
  MAX_CLUSTER_COUNT            = 3
  SCALING_POLICY               = 'ECONOMY'
  MAX_CONCURRENCY_LEVEL        = 8
  STATEMENT_TIMEOUT_IN_SECONDS = 1800
  COMMENT = 'Multi-cluster analyst warehouse | <ENV> | Owner: <TEAM>';
```

### Pattern: Intelligence Warehouse (v2)

```sql
CREATE WAREHOUSE IF NOT EXISTS <ENV>_INTELLIGENCE_WH
  WAREHOUSE_SIZE               = 'SMALL'
  WAREHOUSE_TYPE               = 'SNOWFLAKE_INTELLIGENCE'
  AUTO_SUSPEND                 = 60
  AUTO_RESUME                  = TRUE
  STATEMENT_TIMEOUT_IN_SECONDS = 3600
  COMMENT = 'Cortex AI and Snowflake Intelligence | <ENV> | Owner: <TEAM>';
```

### Pattern: Snowpark-Optimized Warehouse

```sql
CREATE WAREHOUSE IF NOT EXISTS <ENV>_SNOWPARK_WH
  WAREHOUSE_SIZE               = 'MEDIUM'
  WAREHOUSE_TYPE               = 'SNOWPARK-OPTIMIZED'
  AUTO_SUSPEND                 = 120
  AUTO_RESUME                  = TRUE
  MAX_CONCURRENCY_LEVEL        = 2
  STATEMENT_TIMEOUT_IN_SECONDS = 3600
  COMMENT = 'Snowpark Python workloads | <ENV> | Owner: <TEAM>';
```

### Pattern: ML Training Warehouse

```sql
CREATE WAREHOUSE IF NOT EXISTS <ENV>_ML_TRAINING_WH
  WAREHOUSE_SIZE               = 'LARGE'
  WAREHOUSE_TYPE               = 'SNOWPARK-OPTIMIZED'
  AUTO_SUSPEND                 = 120
  AUTO_RESUME                  = TRUE
  MAX_CONCURRENCY_LEVEL        = 1
  STATEMENT_TIMEOUT_IN_SECONDS = 14400
  COMMENT = 'ML training only | <ENV> | Owner: <TEAM>
             | Approved size: LARGE | Approval: <DATE>';
```

## Resource Monitor Pattern

```sql
USE ROLE ACCOUNTADMIN;

CREATE RESOURCE MONITOR IF NOT EXISTS <ENV>_<WH_NAME>_MONITOR
  WITH CREDIT_QUOTA = <QUOTA>
  FREQUENCY = MONTHLY
  START_TIMESTAMP = IMMEDIATELY
  TRIGGERS
    ON 75  PERCENT DO NOTIFY
    ON 90  PERCENT DO NOTIFY
    ON 100 PERCENT DO SUSPEND
    ON 110 PERCENT DO SUSPEND_IMMEDIATE;

ALTER WAREHOUSE <ENV>_<WH_NAME>_WH
  SET RESOURCE_MONITOR = <ENV>_<WH_NAME>_MONITOR;
```

## Database Patterns

### Pattern: Permanent Production Database

```sql
USE ROLE SYSADMIN;
CREATE DATABASE IF NOT EXISTS <ENV>_<DB>_DB
  DATA_RETENTION_TIME_IN_DAYS = <RETENTION>
  COMMENT = '<PURPOSE> | <ENV> | Owner: <TEAM>';
```

### Pattern: Transient Dev/Sandbox Database

```sql
USE ROLE SYSADMIN;
CREATE TRANSIENT DATABASE IF NOT EXISTS <ENV>_<DB>_DB
  DATA_RETENTION_TIME_IN_DAYS = 0
  COMMENT = '<PURPOSE> | <ENV> | Transient | Owner: <TEAM>';
```

## Schema Patterns

### Pattern: Managed Access Schema

```sql
CREATE SCHEMA IF NOT EXISTS <ENV>_<DB>_DB.<SCHEMA>
  WITH MANAGED ACCESS
  DATA_RETENTION_TIME_IN_DAYS = <RETENTION>
  COMMENT = '<PURPOSE> | <ENV> | Managed Access | Owner: <TEAM>';
```

### Pattern: Transient Schema

```sql
CREATE TRANSIENT SCHEMA IF NOT EXISTS <ENV>_<DB>_DB.<SCHEMA>
  DATA_RETENTION_TIME_IN_DAYS = 1
  COMMENT = '<PURPOSE> | <ENV> | Transient | Owner: <TEAM>';
```

## RBAC Architecture

### Role Hierarchy Model

```
ACCOUNTADMIN
  └── SYSADMIN
        ├── <ENV>_DATA_ENGINEER_FR
        │     ├── <ENV>_<DB>_<SCHEMA>_ADMIN_AR
        │     └── <ENV>_ENGINEER_WH_USAGE_AR
        ├── <ENV>_ANALYST_FR
        │     ├── <ENV>_<DB>_<SCHEMA>_READ_AR
        │     └── <ENV>_ANALYST_WH_USAGE_AR
        └── <ENV>_ETL_SVC_FR
              ├── <ENV>_<DB>_<SCHEMA>_WRITE_AR
              └── <ENV>_ETL_WH_USAGE_AR
  └── SECURITYADMIN
        └── USERADMIN
```

### Privilege Level Reference

| Level  | Privileges                                     | Use Case              |
|--------|------------------------------------------------|-----------------------|
| READ   | USAGE on DB/SCHEMA + SELECT on tables/views    | Analysts, BI          |
| WRITE  | READ + INSERT, UPDATE, DELETE                   | ETL, engineers        |
| ADMIN  | WRITE + CREATE TABLE/VIEW/STAGE/etc + TRUNCATE  | Schema admins         |

## Access Role Patterns

### Pattern: Read-Only Access Role

```sql
USE ROLE USERADMIN;
CREATE ROLE IF NOT EXISTS <AR_READ>
  COMMENT = 'Read access to <DB>.<SCHEMA> | Owner: <TEAM>';

USE ROLE SECURITYADMIN;
GRANT USAGE ON DATABASE <DB> TO ROLE <AR_READ>;
GRANT USAGE ON SCHEMA <DB>.<SCHEMA> TO ROLE <AR_READ>;
GRANT SELECT ON ALL TABLES IN SCHEMA <DB>.<SCHEMA> TO ROLE <AR_READ>;
GRANT SELECT ON ALL VIEWS IN SCHEMA <DB>.<SCHEMA> TO ROLE <AR_READ>;
GRANT SELECT ON FUTURE TABLES IN SCHEMA <DB>.<SCHEMA> TO ROLE <AR_READ>;
GRANT SELECT ON FUTURE VIEWS IN SCHEMA <DB>.<SCHEMA> TO ROLE <AR_READ>;
```

### Pattern: Read-Write Access Role

```sql
USE ROLE USERADMIN;
CREATE ROLE IF NOT EXISTS <AR_WRITE>
  COMMENT = 'Write access to <DB>.<SCHEMA> | Owner: <TEAM>';

USE ROLE SECURITYADMIN;
GRANT USAGE ON DATABASE <DB> TO ROLE <AR_WRITE>;
GRANT USAGE ON SCHEMA <DB>.<SCHEMA> TO ROLE <AR_WRITE>;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA <DB>.<SCHEMA> TO ROLE <AR_WRITE>;
GRANT SELECT ON ALL VIEWS IN SCHEMA <DB>.<SCHEMA> TO ROLE <AR_WRITE>;
GRANT SELECT, INSERT, UPDATE, DELETE ON FUTURE TABLES IN SCHEMA <DB>.<SCHEMA> TO ROLE <AR_WRITE>;
GRANT SELECT ON FUTURE VIEWS IN SCHEMA <DB>.<SCHEMA> TO ROLE <AR_WRITE>;
```

### Pattern: Admin Access Role

```sql
USE ROLE USERADMIN;
CREATE ROLE IF NOT EXISTS <AR_ADMIN>
  COMMENT = 'Admin access to <DB>.<SCHEMA> | Owner: <TEAM>';

USE ROLE SECURITYADMIN;
GRANT USAGE ON DATABASE <DB> TO ROLE <AR_ADMIN>;
GRANT USAGE ON SCHEMA <DB>.<SCHEMA> TO ROLE <AR_ADMIN>;
GRANT CREATE TABLE, CREATE VIEW, CREATE STAGE, CREATE FILE FORMAT,
      CREATE FUNCTION, CREATE PROCEDURE, CREATE SEQUENCE, CREATE STREAM,
      CREATE TASK, CREATE PIPE ON SCHEMA <DB>.<SCHEMA> TO ROLE <AR_ADMIN>;
GRANT SELECT, INSERT, UPDATE, DELETE, TRUNCATE ON ALL TABLES IN SCHEMA <DB>.<SCHEMA> TO ROLE <AR_ADMIN>;
GRANT SELECT ON ALL VIEWS IN SCHEMA <DB>.<SCHEMA> TO ROLE <AR_ADMIN>;
GRANT SELECT, INSERT, UPDATE, DELETE, TRUNCATE ON FUTURE TABLES IN SCHEMA <DB>.<SCHEMA> TO ROLE <AR_ADMIN>;
GRANT SELECT ON FUTURE VIEWS IN SCHEMA <DB>.<SCHEMA> TO ROLE <AR_ADMIN>;
GRANT USAGE ON FUTURE STAGES IN SCHEMA <DB>.<SCHEMA> TO ROLE <AR_ADMIN>;
GRANT USAGE ON FUTURE FILE FORMATS IN SCHEMA <DB>.<SCHEMA> TO ROLE <AR_ADMIN>;
GRANT USAGE ON FUTURE FUNCTIONS IN SCHEMA <DB>.<SCHEMA> TO ROLE <AR_ADMIN>;
GRANT USAGE ON FUTURE PROCEDURES IN SCHEMA <DB>.<SCHEMA> TO ROLE <AR_ADMIN>;
```

## Functional Role + Binding Pattern

```sql
USE ROLE USERADMIN;
CREATE ROLE IF NOT EXISTS <FR>
  COMMENT = 'Functional role for <TEAM> | <ENV> | Owner: <TEAM>';

USE ROLE SECURITYADMIN;
GRANT ROLE <AR_1> TO ROLE <FR>;
GRANT ROLE <AR_2> TO ROLE <FR>;
GRANT ROLE <WH_AR> TO ROLE <FR>;
GRANT ROLE <FR> TO ROLE SYSADMIN;
```

## Warehouse Role Grant Pattern

```sql
GRANT USAGE   ON WAREHOUSE <WH> TO ROLE <WH_AR>;
GRANT OPERATE ON WAREHOUSE <WH> TO ROLE <WH_AR>;
GRANT MONITOR ON WAREHOUSE <WH> TO ROLE <ENV>_TEAM_LEAD_ROLE;
```

## Cross-Schema Read Pattern

```sql
GRANT ROLE <ENV>_<DB>_<SCHEMA_A>_READ_AR TO ROLE <ENV>_<TEAM>_FR;
```

## Service Account User Pattern

```sql
USE ROLE USERADMIN;
CREATE USER IF NOT EXISTS <ENV>_<SVC>_SVC_USER
  DEFAULT_ROLE      = '<FR>'
  DEFAULT_WAREHOUSE = '<WH>'
  MUST_CHANGE_PASSWORD = FALSE
  DISABLED          = TRUE
  COMMENT = 'Service account | <ENV> | Owner: <TEAM> | Enable after key-pair setup';

USE ROLE SECURITYADMIN;
GRANT ROLE <FR> TO USER <ENV>_<SVC>_SVC_USER;
```

## Cost-Attribution Tags Pattern

```sql
USE ROLE SYSADMIN;
CREATE DATABASE IF NOT EXISTS GOVERNANCE_DB COMMENT = 'Governance objects';
CREATE SCHEMA IF NOT EXISTS GOVERNANCE_DB.TAGS WITH MANAGED ACCESS;

CREATE TAG IF NOT EXISTS GOVERNANCE_DB.TAGS.ENVIRONMENT
  ALLOWED_VALUES 'DEV', 'UAT', 'STG', 'PRD';
CREATE TAG IF NOT EXISTS GOVERNANCE_DB.TAGS.TEAM;
CREATE TAG IF NOT EXISTS GOVERNANCE_DB.TAGS.COST_CENTER;

ALTER DATABASE <DB> SET TAG
  GOVERNANCE_DB.TAGS.ENVIRONMENT = '<ENV>',
  GOVERNANCE_DB.TAGS.TEAM        = '<TEAM>';
ALTER WAREHOUSE <WH> SET TAG
  GOVERNANCE_DB.TAGS.ENVIRONMENT = '<ENV>',
  GOVERNANCE_DB.TAGS.TEAM        = '<TEAM>',
  GOVERNANCE_DB.TAGS.COST_CENTER = '<CC>';
```

## Teardown Script Structure

```sql
-- !! DANGER: DESTROYS <ENV> environment !!
USE ROLE SECURITYADMIN;
REVOKE ALL ON FUTURE TABLES IN SCHEMA <DB>.<SCHEMA> FROM ROLE <AR>;
REVOKE ROLE <AR> FROM ROLE <FR>;
REVOKE ROLE <FR> FROM ROLE SYSADMIN;

USE ROLE USERADMIN;
DROP ROLE IF EXISTS <FR>;
DROP ROLE IF EXISTS <AR>;

USE ROLE ACCOUNTADMIN;
DROP RESOURCE MONITOR IF EXISTS <MONITOR>;

USE ROLE SYSADMIN;
DROP WAREHOUSE IF EXISTS <WH>;
DROP SCHEMA IF EXISTS <DB>.<SCHEMA>;
DROP DATABASE IF EXISTS <DB>;
```

## Future Grant Rules

- Define at schema level, not database level
- Pair future grants with ALL grants (future is not retroactive)
- Include TABLES + VIEWS minimum; add STAGES, FILE FORMATS,
  FUNCTIONS, PROCEDURES for admin roles
- Schema-level future grants override database-level for same type

## Deployment Order

1. SYSADMIN: Governance DB + tags (if enabled)
2. SYSADMIN: Databases (permanent first, transient second)
3. SYSADMIN: Schemas within each database
4. SYSADMIN: Warehouses (v1 first, v2, then Snowpark)
5. USERADMIN: Access roles (AR)
6. USERADMIN: Functional roles (FR)
7. USERADMIN: Service account users
8. SECURITYADMIN: DB + Schema USAGE grants
9. SECURITYADMIN: Object-level grants (ALL + FUTURE)
10. SECURITYADMIN: Cross-schema read grants
11. SECURITYADMIN: AR → FR binding
12. SECURITYADMIN: Warehouse USAGE grants
13. SECURITYADMIN: FR → SYSADMIN rollup
14. SECURITYADMIN: FR → User binding
15. ACCOUNTADMIN: Resource monitors + assign to warehouses
16. ACCOUNTADMIN: Tags → Objects
17. Validation queries
18. Teardown script (separate file)

## RUN Protocol — Bulk Execution

### Step 1: Generate .sql file
ALWAYS write the full script to a file first.

### Step 2: Execute in batched role blocks
Group ALL statements per role into ONE multi-statement call:

```
Batch 1 → USE ROLE SYSADMIN; CREATE DATABASE...; CREATE SCHEMA...; CREATE WAREHOUSE...;
Batch 2 → USE ROLE USERADMIN; CREATE ROLE...; CREATE USER...;
Batch 3 → USE ROLE SECURITYADMIN; GRANT...; GRANT ROLE...;
Batch 4 → USE ROLE ACCOUNTADMIN; CREATE RESOURCE MONITOR...; ALTER WAREHOUSE...; ALTER DATABASE SET TAG...;
```

4 approvals total per environment. Ask confirmation between envs.

### Failure Handling
- On failure, identify the failing statement, fix, re-run that batch only
- The .sql file remains for manual recovery

## Validation Queries

```sql
SHOW SCHEMAS IN DATABASE <DB>;
SHOW WAREHOUSES LIKE '<ENV>%';
SHOW ROLES LIKE 'AR_%';
SHOW ROLES LIKE 'FR_%';
SHOW RESOURCE MONITORS;
SHOW FUTURE GRANTS IN SCHEMA <DB>.<SCHEMA>;
SHOW GRANTS TO ROLE <FR>;
```

## Anti-Patterns

| Anti-Pattern                        | Correct Approach                              |
|-------------------------------------|-----------------------------------------------|
| Granting privs directly to users    | Use roles                                     |
| ACCOUNTADMIN to create objects      | Use SYSADMIN                                  |
| Skipping future grants              | Always set up for scalability                 |
| No AUTO_SUSPEND on warehouse        | Always set ≥ 60 seconds                       |
| LARGE warehouse without approval    | Default down, document exceptions             |
| Single monolithic role              | Separate AR per DB/schema/privilege            |
| Missing USAGE on parent DB/schema   | Grant full chain: DB → Schema → Object        |
| No resource monitor on warehouse    | Always assign one                             |
| Sharing monitors across warehouses  | One monitor per warehouse                     |
| No COMMENT on objects               | Always document purpose, env, owner           |

# Examples

## Example 1: Full Stack PROD

**User:** Full PROD setup. Database: EDW (INGESTION, TRANSFORM,
ANALYTICS). Teams: DATA_ENGINEER (admin), ANALYST (read ANALYTICS),
ETL_SERVICE (write INGESTION + TRANSFORM). Include warehouses,
monitors, tags.

**Expected:** 1 DB, 3 schemas (managed access), 3 warehouses
(ETL=SMALL, ENGINEER=SMALL, ANALYST=SMALL), 3 resource monitors,
6 access roles, 3 functional roles, 1 SVC user, tags, teardown.

## Example 2: Minimal DEV

**User:** Minimal DEV. One transient DB, one schema, one role, one
XSMALL warehouse. Keep costs under 20 credits/month.

**Expected:** 1 transient DB, 1 schema, 1 XSMALL WH, 1 monitor
(20 credit cap), 1 admin AR, 1 FR, SYSADMIN rollup.

## Example 3: Multi-Env Enterprise

**User:** DEV + UAT + PRD. EDW database, 3 schemas each. 4 teams.
Full RBAC, monitors, tags, service accounts, teardown.

**Expected:** 3 DBs, 9 schemas, 12 warehouses, 12 monitors,
~30 ARs, 12 FRs, 3 SVC users, full tags, teardown script.
Deployed env-by-env with validation between each.

## Example 4: Cortex AI + Snowpark

**User:** Set up compute for Cortex AI workloads and Snowpark
Python pipelines in PRD.

**Expected:** 1 v2 Intelligence WH (Cortex), 1 Snowpark-optimized
WH (Python), resource monitors, warehouse usage ARs.
