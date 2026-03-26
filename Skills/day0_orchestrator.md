---
name: day0-orchestrator
description: Master orchestrator that runs day0-allobjects,
  day0-governance, day0-storage, and day0-pipeline skills in
  sequence. Handles dependency ordering, shared context passing,
  validation checkpoints between phases, and generates a unified
  deployment manifest. Invoke this skill for a complete Day 0
  Snowflake environment build-out.
tools:
  - snowflake_sql_execute
  - snowflake_object_search
  - ask_user_question
---

# When to Use

- Full Day 0 environment build-out (end-to-end)
- Customer says "set up everything" or "full environment"
- Need to run multiple day0 skills in correct dependency order
- Want a single entry point that orchestrates all phases

# What This Skill Provides

A sequenced execution framework that runs four sub-skills in
dependency order, passing shared context between phases, with
validation checkpoints after each phase. One skill to rule them all.

# Architecture

```
/day0-orchestrator (this skill)
  │
  ├── Phase 1: /day0-allobjects
  │     DBs, schemas, warehouses, RBAC, monitors, tags
  │     OUTPUT → database names, schema names, role names, WH names
  │
  ├── Phase 2: /day0-governance
  │     Classification profiles, PII tags, masking policies, RAPs
  │     DEPENDS ON → governance schema, exempt roles from Phase 1
  │
  ├── Phase 3: /day0-storage
  │     Table design, clustering, search optimization, column governance
  │     DEPENDS ON → schemas, governance tags/policies from Phase 1+2
  │
  └── Phase 4: /day0-pipeline
        Stages, file formats, COPY INTO, Snowpipe, streams, tasks, DTs
        DEPENDS ON → tables from Phase 3, warehouses from Phase 1
```

# Instructions

## Pre-Flight: Scope Discovery

Before running ANY phase, gather the master configuration
that feeds ALL sub-skills. Ask ONCE, use everywhere.

### Master Configuration Questions

1. **Environments**: DEV, UAT, STG, PRD? (feeds all phases)
2. **Naming convention**: ENV_NAME_DB or NAME_ENV_DB?
3. **Owner team**: Who owns this infrastructure?
4. **Databases + schemas**: List all databases and schemas
5. **Teams + access levels**: Who needs what access?
6. **Warehouse strategy**: One per team? Per workload?
7. **Compliance frameworks**: GDPR, CCPA, HIPAA, PCI-DSS?
8. **Data sources**: What systems will feed data in?
   (S3, Azure Blob, GCS, internal, streaming?)
9. **Table patterns needed**: Dimension, fact, landing, SCD2?
10. **Pipeline patterns**: Batch COPY, Snowpipe, CDC streams,
    dynamic tables?

STOPPING POINT: Present the unified manifest covering all 4 phases.
User must confirm before any execution begins.

### Manifest Template

Present a single summary table:

```
=== DAY 0 DEPLOYMENT MANIFEST ===

ENVIRONMENTS: DEV, PRD
NAMING:       NAME_ENV_DB
OWNER:        DATA_ENGINEERING

PHASE 1 — INFRASTRUCTURE (day0-allobjects)
  Databases:    EDW_DEV_DB, EDW_PRD_DB
  Schemas:      INGESTION, TRANSFORM, ANALYTICS
  Warehouses:   DEV_ETL_WH(XS), PRD_ETL_WH(S), ...
  Roles:        FR_DATA_ENGINEER, FR_ANALYST, FR_ETL_SVC
  Monitors:     Per-warehouse, 20/150 credit quotas

PHASE 2 — GOVERNANCE (day0-governance)
  Gov Schema:   AUDIT_PRD_DB.GOVERNANCE
  Tags:         PII_TYPE, SENSITIVITY, COMPLIANCE
  Masking:      Conditional tag-based (VARCHAR, NUMBER, DATE, TS)
  RAP:          Role-based with mapping table
  Classification: Auto-classify on all schemas
  Exempt Roles: GOVERNANCE_ADMIN, FR_DATA_ENGINEER_PRD

PHASE 3 — STORAGE (day0-storage)
  Tables:       CUSTOMER (dim), ORDER (fact), EVENT_LANDING (raw)
  Clustering:   ORDER fact on (EVENT_DATE, REGION)
  Governance:   PII tags + masking on name/email/phone/DOB

PHASE 4 — PIPELINES (day0-pipeline)
  Integrations: S3 storage integration
  Stages:       External S3 stage for events
  Formats:      CSV, JSON, Parquet
  Pipes:        Snowpipe auto-ingest for events
  Streams:      CDC on CUSTOMER landing table
  Tasks:        Hourly ETL transform task
  Dynamic Tables: DAILY_REVENUE (1hr lag)
```

## Execution Protocol

### Rule 1: Sequential Phase Execution

Phases MUST run in order 1→2→3→4. Each phase depends on
outputs from the previous phase:

| Phase | Depends On |
|-------|-----------|
| 1. allobjects | Nothing (foundation) |
| 2. governance | Phase 1: governance schema, exempt roles |
| 3. storage | Phase 1: schemas. Phase 2: tags, masking policies |
| 4. pipeline | Phase 1: warehouses. Phase 3: target tables |

### Rule 2: Read Sub-Skill Instructions

For each phase, READ the corresponding SKILL.md file and
follow its instructions, guardrails, and patterns:

```
Phase 1 → Read .cortex/skills/day0-allobjects/SKILL.md
Phase 2 → Read .cortex/skills/day0-governance/SKILL.md
Phase 3 → Read .cortex/skills/day0-storage/SKILL.md
Phase 4 → Read .cortex/skills/day0-pipeline/SKILL.md
```

### Rule 3: Batched RUN Per Phase

Each phase uses the batched RUN protocol (max 4 approvals
per phase). Total approvals for full deployment:

| Phase | Approvals | Role Blocks |
|-------|-----------|-------------|
| 1. allobjects | 4 | SYSADMIN, USERADMIN, SECURITYADMIN, ACCOUNTADMIN |
| 2. governance | 4 | SYSADMIN, USERADMIN, SECURITYADMIN, ACCOUNTADMIN |
| 3. storage | 2 | SYSADMIN (tables), SECURITYADMIN (tags+masking) |
| 4. pipeline | 3 | SYSADMIN (stages+formats), ACCOUNTADMIN (integrations), SECURITYADMIN (grants) |
| **Total** | **~13** | **Per environment** |

### Rule 4: Validation Checkpoint Between Phases

After each phase completes, run validation queries from
that phase's SKILL.md. Present results. Ask:

"Phase N complete. [X objects created, Y grants applied].
Proceed to Phase N+1?"

Do NOT proceed until user confirms.

### Rule 5: Shared Context

Maintain a running context object that accumulates outputs
from each phase. Use this context to feed into subsequent phases:

```
CONTEXT = {
  environments: ['DEV', 'PRD'],
  databases: { 'EDW_DEV_DB': [...schemas], 'EDW_PRD_DB': [...] },
  warehouses: ['DEV_ETL_WH', 'PRD_ETL_WH', ...],
  roles: {
    functional: ['FR_DATA_ENGINEER_DEV', ...],
    access: ['AR_EDW_DEV_INGESTION_ADMIN', ...],
    exempt: ['GOVERNANCE_ADMIN', 'FR_DATA_ENGINEER_PRD']
  },
  governance: {
    schema: 'AUDIT_PRD_DB.GOVERNANCE',
    tags: ['PII_TYPE', 'SENSITIVITY', 'COMPLIANCE'],
    policies: ['PII_TAG_MASK_STRING', ...]
  },
  tables: ['EDW_PRD_DB.ANALYTICS.CUSTOMER', ...],
  pipelines: ['PRD_EVENTS_PIPE', ...]
}
```

### Rule 6: Generate Unified .sql File

After all 4 phases complete, generate a single consolidated
`day0_full_deployment.sql` file containing ALL SQL from all
phases, organized by role blocks. This serves as:
- Documentation / audit trail
- Repeatable deployment for new environments
- Disaster recovery reference

### Rule 7: Skip Phases Selectively

If the user says "skip Phase 3" or "no tables yet", respect
the skip but WARN about downstream dependencies:

"Skipping Phase 3 (storage). Note: Phase 4 (pipelines)
requires target tables. Pipeline SQL will be generated
but NOT executed — you'll need to run it after creating tables."

## Phase-Specific Guardrails

### Phase 1 Guardrails (allobjects)
- Follow all guardrails from day0-allobjects SKILL.md
- Create governance schema here (used by Phase 2)
- Create ALL warehouses (used by Phase 4 tasks/DTs)

### Phase 2 Guardrails (governance)
- Governance schema MUST exist from Phase 1
- Exempt roles MUST exist from Phase 1
- Classification profiles reference schemas from Phase 1
- Tag-based masking binds to tags created in this phase

### Phase 3 Guardrails (storage)
- Tables go into schemas created in Phase 1
- PII columns get tags from Phase 2
- Masking policies from Phase 2 auto-apply via tag binding
- Clustering only on tables > 1TB (flag for future)

### Phase 4 Guardrails (pipeline)
- Stages go into schemas from Phase 1
- COPY INTO targets tables from Phase 3
- Tasks use warehouses from Phase 1
- Snowpipe uses service roles from Phase 1
- Dynamic tables use warehouses from Phase 1

## Anti-Patterns

| Anti-Pattern | Correct Approach |
|---|---|
| Running Phase 2 before Phase 1 | Always run in order 1→2→3→4 |
| Creating tables before governance | Phase 2 (governance) before Phase 3 (storage) |
| Creating pipes before target tables | Phase 3 (tables) before Phase 4 (pipes) |
| Skipping validation between phases | Always validate and checkpoint |
| Not generating unified .sql file | Always produce the consolidated file |
| Asking all discovery questions per phase | Ask ONCE in pre-flight, reuse context |

# Examples

## Example 1: Full Enterprise Deployment

**User:** `/day0-orchestrator` Set up everything for PROD.
EDW database with INGESTION, TRANSFORM, ANALYTICS.
Teams: DATA_ENGINEER, ANALYST, ETL_SERVICE.
Compliance: GDPR. Source: S3 CSV files.
Tables: CUSTOMER (dim), ORDER (fact).
Pipeline: hourly batch load + CDC stream.

**Expected:** 4-phase deployment with ~13 approvals total,
unified day0_full_deployment.sql, all validation passing.

## Example 2: Infrastructure + Governance Only

**User:** `/day0-orchestrator` Set up infra and governance for
DEV + PRD. Skip storage and pipelines — we'll add tables later.

**Expected:** Phase 1 + Phase 2 only. Phase 3 + 4 skipped
with dependency warning. ~8 approvals total.

## Example 3: Add Pipelines to Existing Environment

**User:** `/day0-orchestrator` We already have Phase 1-3 done.
Just add pipelines: Snowpipe from S3 for events, CDC stream
on CUSTOMER table, daily revenue dynamic table.

**Expected:** Skip Phase 1-3 (validate existing objects),
run Phase 4 only. ~3 approvals.
