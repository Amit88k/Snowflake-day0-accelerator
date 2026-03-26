---
name: day0-compute
description: Use when setting up or configuring Snowflake compute
  resources including virtual warehouses (v1 and v2), serverless
  and dynamic compute, ML/Snowpark warehouses, resource monitors,
  and auto-scaling policies for a DAY0 environment setup.
tools:
  - snowflake_sql_execute
  - snowflake_object_search
---

# When to Use

- Customer needs virtual warehouses created for a new environment
- Choosing between Warehouse v1, v2, or serverless compute
- Configuring resource monitors and credit limits
- Setting up auto-suspend and auto-resume policies
- Sizing warehouses for specific workload types
- Configuring ML, Snowpark, or data science compute
- Applying multi-cluster warehouse configuration
- Setting up dynamic or serverless warehouse alternatives

# What This Skill Provides

Complete, production-ready compute setup for a Snowflake environment
following Snowflake-recommended best practices across all warehouse
types. Outputs fully qualified, parameterized SQL ready for human
review before execution.

# Instructions

## Core Guardrails (Non-Negotiable — Always Apply)

- ALWAYS use fully qualified names: DATABASE.SCHEMA.OBJECT
- ALWAYS include AUTO_SUSPEND on every warehouse — NEVER omit
- ALWAYS include AUTO_RESUME = TRUE on every warehouse
- ALWAYS use IF NOT EXISTS on every CREATE statement
- NEVER create a warehouse without a resource monitor assigned
- NEVER set AUTO_SUSPEND below 60 seconds in any environment
- NEVER use WAREHOUSE_SIZE = LARGE or above without explicit
  customer written approval — default down, not up
- ALWAYS add a COMMENT field describing warehouse purpose,
  owner team, and environment
- NEVER grant MODIFY or OWNERSHIP on a warehouse to non-admin roles
- ALWAYS grant USAGE on warehouse to a role, never to a user directly
- NEVER create warehouses with identical names across environments —
  always prefix with environment name: DEV_, STG_, PROD_
- ALWAYS set STATEMENT_TIMEOUT_IN_SECONDS — never leave unbounded
- NEVER enable QUERY_ACCELERATION without reviewing cost implications first

## Warehouse Type Decision Guide

Before creating any warehouse, determine the correct type:

```
Customer workload described?
│
├── Predictable, scheduled batch / ETL?
│   └── Warehouse v1 (standard)
│
├── Interactive, variable, analyst / BI?
│   └── Warehouse v2 (Snowflake Intelligence tier)
│
├── Serverless tasks, pipes, search, replication?
│   └── Serverless compute (no warehouse needed)
│
├── Python, Pandas, Snowpark DataFrame workloads?
│   └── Snowpark-optimized warehouse
│
└── ML training, feature engineering, model inference?
    └── Snowpark-optimized warehouse (LARGE+ justified)
```

## Warehouse v1 — Standard Virtual Warehouse

Use for: ETL, data loading, scheduled batch jobs, admin tasks,
and traditional SQL workloads with predictable patterns.

### Sizing Reference

| Workload              | Size            | Auto-Suspend | Timeout |
|-----------------------|-----------------|--------------|---------|
| Admin / Monitoring    | XSMALL          | 60s          | 1800s   |
| Ad-hoc Analyst        | XSMALL or SMALL | 60s          | 1800s   |
| Reporting / BI        | SMALL           | 60s          | 3600s   |
| ETL / Data Loading    | SMALL or MEDIUM | 120s         | 7200s   |
| Heavy Transformation  | MEDIUM          | 180s         | 7200s   |

### Guardrails Specific to v1

- MAX_CONCURRENCY_LEVEL default is 8 — reduce to 4 for ETL
  to prevent resource contention
- ENABLE_QUERY_ACCELERATION = FALSE by default — only enable
  for long-running selective queries after profiling
- For multi-cluster: MIN = 1 always, MAX = peak_users / 8
  rounded up, SCALING_POLICY = ECONOMY default

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
  SCALING_POLICY               = ECONOMY
  MAX_CONCURRENCY_LEVEL        = 8
  STATEMENT_TIMEOUT_IN_SECONDS = 1800
  COMMENT = 'Multi-cluster analyst warehouse | <ENV> | Owner: <TEAM>';
```

## Warehouse v2 — Snowflake Intelligence Warehouse

Use for: Variable, interactive workloads where query patterns are unpredictable.
Optimized for Cortex AI, Snowflake Intelligence, and mixed analyst/AI workloads.

### When to Recommend v2 Over v1

- Workload includes Cortex AI function calls (COMPLETE, CLASSIFY, etc.)
- Mixed analyst + AI/ML inference in same session
- Customer uses Snowflake Intelligence or Cortex Analyst heavily
- Query duration is highly variable (seconds to minutes in same WH)

### Guardrails Specific to v2

- WAREHOUSE_TYPE = 'SNOWFLAKE_INTELLIGENCE'
- Still requires AUTO_SUSPEND and resource monitor — same rules apply
- Do NOT use v2 for pure ETL/batch — v1 is more cost-effective there
- ALWAYS document why v2 was chosen in the COMMENT field

### Pattern: Cortex AI / Intelligence Warehouse (v2)

```sql
CREATE WAREHOUSE IF NOT EXISTS <ENV>_INTELLIGENCE_WH
  WAREHOUSE_SIZE               = 'SMALL'
  WAREHOUSE_TYPE               = 'SNOWFLAKE_INTELLIGENCE'
  AUTO_SUSPEND                 = 60
  AUTO_RESUME                  = TRUE
  STATEMENT_TIMEOUT_IN_SECONDS = 3600
  COMMENT = 'Cortex AI and Snowflake Intelligence workloads | <ENV> | Owner: <TEAM>';
```

## Snowpark-Optimized Warehouse

Use for: Python workloads using Snowpark DataFrames, pandas on Snowflake,
UDFs with large memory requirements, and any workload that processes data
in-memory at scale.

> Snowpark-optimized nodes have 16x memory per node compared to standard
> warehouses — critical for avoiding out-of-memory errors in Python workloads.

### Guardrails Specific to Snowpark

- WAREHOUSE_TYPE = 'SNOWPARK-OPTIMIZED' — must be explicit
- Minimum effective size is MEDIUM — XSMALL/SMALL rarely justified
- AUTO_SUSPEND = 120 minimum — Snowpark sessions take longer to init
- NEVER use for SQL-only workloads — cost is higher per credit
- ALWAYS pair with a dedicated resource monitor — credits burn faster
- STATEMENT_TIMEOUT_IN_SECONDS = 14400 for training jobs, 3600 for general Snowpark use
- If used for ML training: document expected job duration in COMMENT
- LARGE size requires explicit customer approval — record in COMMENT

### Pattern: Snowpark General Purpose Warehouse

```sql
CREATE WAREHOUSE IF NOT EXISTS <ENV>_SNOWPARK_WH
  WAREHOUSE_SIZE               = 'MEDIUM'
  WAREHOUSE_TYPE               = 'SNOWPARK-OPTIMIZED'
  AUTO_SUSPEND                 = 120
  AUTO_RESUME                  = TRUE
  MAX_CONCURRENCY_LEVEL        = 2
  STATEMENT_TIMEOUT_IN_SECONDS = 3600
  COMMENT = 'Snowpark Python DataFrame workloads | <ENV> | Owner: <TEAM>';
```

### Pattern: ML Training Warehouse (Snowpark-Optimized)

```sql
CREATE WAREHOUSE IF NOT EXISTS <ENV>_ML_TRAINING_WH
  WAREHOUSE_SIZE               = 'LARGE'
  WAREHOUSE_TYPE               = 'SNOWPARK-OPTIMIZED'
  AUTO_SUSPEND                 = 120
  AUTO_RESUME                  = TRUE
  MAX_CONCURRENCY_LEVEL        = 1
  STATEMENT_TIMEOUT_IN_SECONDS = 14400
  COMMENT = 'ML model training jobs only | <ENV> | Owner: <DATA_SCIENCE_TEAM>
             | Approved size: LARGE | Approval date: <DATE>';
```

## Serverless Compute

Use for: Snowflake Tasks, Snowpipe, Search Optimization, Automatic Clustering,
Replication, and Cortex AI SQL functions when called outside a warehouse context.

### Guardrails for Serverless

- NEVER assign a warehouse to serverless tasks
- Monitor serverless credit usage via SERVERLESS_TASK_HISTORY in ACCOUNT_USAGE —
  it does NOT appear in warehouse monitors
- Create a dedicated budget object to cap serverless spend

### Pattern: Serverless Budget Object

```sql
CREATE BUDGET IF NOT EXISTS <ENV>_SERVERLESS_BUDGET
  WITH CREDIT_QUOTA = 50
  FREQUENCY        = MONTHLY
  START_TIMESTAMP  = IMMEDIATELY
  NOTIFY_USERS     = ('<OWNER_USER>')
  TRIGGERS
    ON 75  PERCENT DO NOTIFY
    ON 100 PERCENT DO SUSPEND_IMMEDIATE;
```

## Resource Monitor Rules (All Warehouse Types)

- Create ONE resource monitor per warehouse — never share across unrelated warehouses
- ALWAYS set minimum two triggers: NOTIFY at 75%, SUSPEND at 100%
- Add NOTIFY at 90% for production warehouses
- FREQUENCY = MONTHLY default; WEEKLY for high-burn warehouses
- After creating monitor, ALWAYS ALTER WAREHOUSE to assign it

### Pattern: Full Resource Monitor (Production Standard)

```sql
CREATE RESOURCE MONITOR IF NOT EXISTS <ENV>_<WH_NAME>_MONITOR
  WITH CREDIT_QUOTA = <QUOTA>
  FREQUENCY        = MONTHLY
  START_TIMESTAMP  = IMMEDIATELY
  NOTIFY_USERS     = ('<OWNER_USER>', '<BACKUP_USER>')
  TRIGGERS
    ON 75  PERCENT DO NOTIFY
    ON 90  PERCENT DO NOTIFY
    ON 100 PERCENT DO SUSPEND
    ON 110 PERCENT DO SUSPEND_IMMEDIATE;

ALTER WAREHOUSE <ENV>_<WH_NAME>_WH
  SET RESOURCE_MONITOR = <ENV>_<WH_NAME>_MONITOR;
```

## Credit Quota Reference

| Size   | Standard v1    | Snowpark-Optimized | Suggested Monthly Quota |
|--------|----------------|--------------------|-------------------------|
| XSMALL | 1 credit/hr    | N/A                | 20–50                    |
| SMALL  | 2 credits/hr   | N/A                | 50–150                   |
| MEDIUM | 4 credits/hr   | 6 credits/hr       | 100–300                  |
| LARGE  | 8 credits/hr   | 12 credits/hr      | 200–500                  |

Use lower bound for dev/test, upper bound for production.
Always confirm quota with customer before setting.

## Role Grant Rules

After creating warehouses, ALWAYS generate grants in this pattern:

- USAGE → functional role (ANALYST_ROLE, ETL_ROLE, etc.)
- OPERATE → role that needs to start/stop warehouse
- MONITOR → team lead or cost owner role
- MODIFY → restricted to admin roles ONLY
- NEVER grant ALL PRIVILEGES on a warehouse

### Pattern: Warehouse Role Grants

```sql
GRANT USAGE   ON WAREHOUSE <ENV>_<WH>_WH TO ROLE <ENV>_<FUNCTIONAL_ROLE>;
GRANT OPERATE ON WAREHOUSE <ENV>_<WH>_WH TO ROLE <ENV>_<FUNCTIONAL_ROLE>;
GRANT MONITOR ON WAREHOUSE <ENV>_<WH>_WH TO ROLE <ENV>_TEAM_LEAD_ROLE;
```

## Deployment Order

Always generate and deploy in this exact sequence:

1. Resource monitors (must exist before warehouses reference them)
2. Warehouses v1 (standard)
3. Warehouses v2 (intelligence)
4. Warehouses Snowpark-optimized
5. Assign resource monitors to all warehouses
6. Grant USAGE and OPERATE to functional roles
7. Serverless budget objects
8. Validation queries

## Validation Queries

Always append these after all deployment scripts:

```sql
-- Confirm all warehouses created with correct configuration
SELECT NAME, SIZE, TYPE, AUTO_SUSPEND, AUTO_RESUME,
       RESOURCE_MONITOR, COMMENT
FROM   INFORMATION_SCHEMA.WAREHOUSES
WHERE  NAME LIKE '<ENV>%'
ORDER  BY NAME;

-- Confirm resource monitors are assigned
SELECT NAME, CREDIT_QUOTA, FREQUENCY, OWNER
FROM   SNOWFLAKE.ACCOUNT_USAGE.RESOURCE_MONITORS
WHERE  NAME LIKE '<ENV>%';

-- Confirm role grants on warehouses
SHOW GRANTS ON WAREHOUSE <ENV>_<WH_NAME>_WH;
```

# Examples

## Example 1: Full Compute Stack for PROD

**User:** `$day0-compute` Create a full compute stack for PROD.
Workloads: analyst queries, nightly ETL, Snowpark Python
pipelines, and ML training. Monthly budget per warehouse:
100 credits. Owner: DATA_PLATFORM_TEAM.

**Expected output:** 4 warehouses (ANALYST v1, ETL v1,
SNOWPARK general, ML training Snowpark-optimized),
4 resource monitors, all role grants, validation queries.

## Example 2: Cortex AI Workload

**User:** `$day0-compute` Set up a warehouse for Cortex AI
function calls and Snowflake Intelligence queries in DEV.
Budget: 30 credits monthly.

**Expected output:** v2 Intelligence warehouse, resource monitor,
role grants, serverless budget for Cortex usage.

## Example 3: Cost-Constrained Dev Environment

**User:** `$day0-compute` Minimal compute setup for DEV.
One warehouse only. Keep costs under 20 credits/month.

**Expected output:** Single XSMALL v1 warehouse, 20-credit
monitor with SUSPEND at 100%, USAGE grant to DEV role.
