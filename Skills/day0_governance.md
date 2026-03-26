---
name: day0-governance
description: Use when identifying PII/PHI data through classification
  profiles, applying object tags, creating dynamic data masking
  policies, row access policies, and tag-based masking for a
  Snowflake environment. Covers SYSTEM$CLASSIFY, automatic
  classification, custom classifiers, semantic/privacy categories,
  masking policy patterns (full, partial, email, SHA-256, conditional),
  row-level security, governance auditing, and compliance workflows
  (GDPR, CCPA, HIPAA, PCI-DSS). Supports bulk RUN execution.
tools:
  - snowflake_sql_execute
  - snowflake_object_search
  - ask_user_question
---

# When to Use

- Customer needs to identify PII/PHI in existing tables
- Running SYSTEM$CLASSIFY on schemas or databases
- Setting up automatic classification with Classification Profiles
- Creating custom classifiers (regex, value patterns)
- Applying semantic/privacy tags to columns
- Creating dynamic data masking policies
- Setting up tag-based masking (policy auto-applies via tag)
- Creating row access policies for row-level security
- Building compliance workflows (GDPR, CCPA, HIPAA, PCI-DSS)
- Auditing which columns are masked and which roles are exempt
- Choosing between masking vs. row access vs. both

# What This Skill Provides

End-to-end data governance deployment: discover sensitive data via
classification, tag columns with semantic/privacy categories, apply
masking policies that auto-trigger via tags, and add row-level
security where needed. All SQL is idempotent and follows Snowflake
recommended best practices. Supports batched RUN execution
(4 approvals max).

# Instructions

## Interactive Discovery Protocol

### Phase 1: Scope Discovery

Ask the user for:
1. **Target objects**: Which databases/schemas/tables to scan?
2. **Governance database**: Where to store tags and policies?
   (Default: GOVERNANCE_DB.POLICIES)
3. **Compliance requirements**: GDPR, CCPA, HIPAA, PCI-DSS, or general?
4. **Existing tags/policies**: Any already in place?

STOPPING POINT: Confirm scope before scanning.

### Phase 2: Classification

Run SYSTEM$CLASSIFY on target objects and present findings:
1. **Review results**: Show discovered PII/PHI columns
2. **Confirm accuracy**: User validates classifications
3. **Custom classifiers**: Need to detect custom patterns?
   (e.g., internal IDs, account numbers, MRN)

STOPPING POINT: Confirm classification results before tagging.

### Phase 3: Protection Strategy

For each classified column, determine protection method:
1. **Masking candidates**: Which columns need dynamic masking?
2. **Row-level security**: Any tables need row-level filtering?
3. **Exempt roles**: Which roles see unmasked data?
4. **Masking style**: Full mask, partial, hash, email domain, etc.

STOPPING POINT: Present protection matrix. User must confirm.

### Phase 4: Optional Add-Ons

1. **Automatic classification**: Set up Classification Profile
   for ongoing auto-detection on new tables?
2. **Tag-based masking**: Bind masking policies to tags so new
   columns auto-mask when tagged? (Recommended: YES)
3. **Governance audit view**: Create a view showing all masked
   columns and their policies?
4. **Teardown script**: Generate rollback script?

## Core Guardrails (Non-Negotiable)

- ALWAYS create tags and policies in a dedicated governance
  schema (e.g., GOVERNANCE_DB.POLICIES) — never in data schemas
- ALWAYS use ACCOUNTADMIN or a role with APPLY MASKING POLICY
  and APPLY TAG privileges for policy/tag assignments
- ALWAYS use CREATE OR REPLACE for masking policies during
  initial development, then switch to IF NOT EXISTS for production
- NEVER apply masking policies directly to columns when tag-based
  masking is available — use tag-based masking for scalability
- ALWAYS include a WHEN clause in masking policies that checks
  CURRENT_ROLE() or IS_ROLE_IN_SESSION() for role exemptions
- NEVER use CURRENT_ROLE() alone for role checks — use
  IS_ROLE_IN_SESSION() to respect role hierarchy inheritance
- ALWAYS test masking policies with a non-exempt role before
  declaring success
- NEVER mask primary keys or join keys — this breaks queries
- NEVER apply row access policies AND masking policies that
  reference the same column — this causes infinite recursion
- ALWAYS grant APPLY MASKING POLICY and APPLY TAG to a
  dedicated governance role — never to SYSADMIN directly
- ALWAYS set MASKING_POLICY_EXEMPT_OTHER_POLICIES = FALSE
  unless explicitly needed
- ALWAYS document the exempt roles list in the policy COMMENT
- NEVER create masking policies that return different data TYPES
  than the input column — this causes runtime errors
- ALWAYS handle NULL inputs in masking policies
- ALWAYS use EXEMPT_OTHER_POLICIES = FALSE by default on row
  access policies — only use TRUE with documented justification
- WHEN choosing between masking and row access:
  - Use MASKING when you need to hide column VALUES but keep rows
  - Use ROW ACCESS when you need to hide entire ROWS
  - Use BOTH when some columns need masking AND some rows need
    filtering (on DIFFERENT columns)

## Protection Strategy Decision Guide

```
What needs protecting?
│
├── Column values need hiding (but rows stay visible)?
│   └── DYNAMIC DATA MASKING
│       │
│       ├── One-off columns (< 10)?
│       │   └── Direct column masking policy
│       │
│       └── Many columns or ongoing tagging?
│           └── TAG-BASED MASKING (recommended)
│               Policy auto-applies when tag is set
│
├── Entire rows need filtering by viewer's role/region/dept?
│   └── ROW ACCESS POLICY
│       │
│       ├── Filter by role (e.g., regional data access)?
│       │   └── Role-based RAP with mapping table
│       │
│       └── Filter by attribute (e.g., department, country)?
│           └── Attribute-based RAP with mapping table
│
└── Both column masking AND row filtering needed?
    └── Use BOTH — but on DIFFERENT columns
        Masking on sensitive values (SSN, email)
        Row access on filtering dimension (region, dept)
```

## Classification Patterns

### Pattern: Manual Classification (Single Table)

```sql
SELECT SYSTEM$CLASSIFY('<DB>.<SCHEMA>.<TABLE>',
  {'auto_tag': false, 'sample_count': 10000});
```

Returns JSON with column-level classifications including:
- `semantic_category`: NAME, EMAIL, PHONE, IP_ADDRESS, etc.
- `privacy_category`: IDENTIFIER, QUASI_IDENTIFIER, SENSITIVE
- `confidence`: HIGH, MEDIUM, LOW (only act on HIGH)

### Pattern: Manual Classification (Entire Schema)

```sql
SELECT t.TABLE_NAME,
       SYSTEM$CLASSIFY('<DB>.<SCHEMA>.' || t.TABLE_NAME,
         {'auto_tag': false, 'sample_count': 10000}) AS classification
FROM <DB>.INFORMATION_SCHEMA.TABLES t
WHERE t.TABLE_SCHEMA = '<SCHEMA>'
  AND t.TABLE_TYPE = 'BASE TABLE';
```

### Pattern: Classification with Auto-Tagging

```sql
SELECT SYSTEM$CLASSIFY('<DB>.<SCHEMA>.<TABLE>',
  {'auto_tag': true, 'sample_count': 10000});
```

This automatically applies Snowflake system tags:
- `SNOWFLAKE.CORE.SEMANTIC_CATEGORY`
- `SNOWFLAKE.CORE.PRIVACY_CATEGORY`

### Pattern: Automatic Classification via Profile

```sql
CREATE OR REPLACE DATA PRIVACY CLASSIFICATION PROFILE
  <DB>.<SCHEMA>.<PROFILE_NAME>
  CONFIG = '{
    "auto_tag": true,
    "minimum_object_age_for_classification_days": 0,
    "maximum_classification_validity_days": 30
  }'
  COMMENT = 'Auto-classifies new tables in <SCHEMA> | Owner: <TEAM>';

ALTER SCHEMA <DB>.<SCHEMA>
  SET CLASSIFICATION PROFILE = '<DB>.<SCHEMA>.<PROFILE_NAME>';
```

### Pattern: Custom Classifier (Regex)

For detecting patterns not covered by built-in classifiers
(e.g., internal IDs, medical record numbers):

```sql
CREATE OR REPLACE SNOWFLAKE.DATA_PRIVACY.CUSTOM_CLASSIFIER
  <GOVERNANCE_DB>.<GOVERNANCE_SCHEMA>.<CLASSIFIER_NAME>()
RETURNS TABLE(col_name VARCHAR)
LANGUAGE SQL
AS $$
  SELECT col_name FROM (
    SELECT $1 AS col_name FROM VALUES
    ('<COLUMN_NAME>')
  )
  WHERE REGEXP_LIKE(col_name, '<REGEX_PATTERN>')
$$;
```

## Tag Patterns

### Pattern: Governance Tag Schema

```sql
USE ROLE SYSADMIN;
CREATE DATABASE IF NOT EXISTS GOVERNANCE_DB
  COMMENT = 'Centralized governance objects';
CREATE SCHEMA IF NOT EXISTS GOVERNANCE_DB.POLICIES
  WITH MANAGED ACCESS
  COMMENT = 'Tags, masking policies, and row access policies';

CREATE TAG IF NOT EXISTS GOVERNANCE_DB.POLICIES.PII_TYPE
  ALLOWED_VALUES 'EMAIL', 'PHONE', 'SSN', 'NAME', 'ADDRESS',
    'DOB', 'IP_ADDRESS', 'CREDIT_CARD', 'PASSPORT',
    'MEDICAL_RECORD', 'NONE'
  COMMENT = 'PII classification type for tag-based masking';

CREATE TAG IF NOT EXISTS GOVERNANCE_DB.POLICIES.SENSITIVITY
  ALLOWED_VALUES 'PUBLIC', 'INTERNAL', 'CONFIDENTIAL', 'RESTRICTED'
  COMMENT = 'Data sensitivity level';

CREATE TAG IF NOT EXISTS GOVERNANCE_DB.POLICIES.COMPLIANCE
  ALLOWED_VALUES 'GDPR', 'CCPA', 'HIPAA', 'PCI_DSS', 'NONE'
  COMMENT = 'Applicable compliance framework';
```

### Pattern: Apply Tags to Classified Columns

```sql
ALTER TABLE <DB>.<SCHEMA>.<TABLE>
  MODIFY COLUMN <COLUMN>
  SET TAG GOVERNANCE_DB.POLICIES.PII_TYPE = '<TYPE>';

ALTER TABLE <DB>.<SCHEMA>.<TABLE>
  MODIFY COLUMN <COLUMN>
  SET TAG GOVERNANCE_DB.POLICIES.SENSITIVITY = 'CONFIDENTIAL';
```

## Masking Policy Patterns

### Guardrails for Masking Policies

- Input and output data types MUST match exactly
- ALWAYS handle NULLs (return NULL for NULL input)
- ALWAYS use IS_ROLE_IN_SESSION() not CURRENT_ROLE()
- Create ONE policy per data type per masking style
- NEVER create per-column policies — use tag-based masking

### Pattern: Full Mask (VARCHAR)

```sql
CREATE OR REPLACE MASKING POLICY GOVERNANCE_DB.POLICIES.FULL_MASK_STRING
  AS (val VARCHAR) RETURNS VARCHAR ->
  CASE
    WHEN IS_ROLE_IN_SESSION('GOVERNANCE_ADMIN') THEN val
    WHEN IS_ROLE_IN_SESSION('<EXEMPT_ROLE>') THEN val
    WHEN val IS NULL THEN NULL
    ELSE '********'
  END
  COMMENT = 'Full string mask | Exempt: GOVERNANCE_ADMIN, <EXEMPT_ROLE>';
```

### Pattern: Email Mask (show domain only)

```sql
CREATE OR REPLACE MASKING POLICY GOVERNANCE_DB.POLICIES.EMAIL_MASK
  AS (val VARCHAR) RETURNS VARCHAR ->
  CASE
    WHEN IS_ROLE_IN_SESSION('GOVERNANCE_ADMIN') THEN val
    WHEN IS_ROLE_IN_SESSION('<EXEMPT_ROLE>') THEN val
    WHEN val IS NULL THEN NULL
    ELSE CONCAT('*****@', SPLIT_PART(val, '@', 2))
  END
  COMMENT = 'Email domain-only mask | Exempt: GOVERNANCE_ADMIN, <EXEMPT_ROLE>';
```

### Pattern: Partial Mask — Last 4 (SSN / Credit Card)

```sql
CREATE OR REPLACE MASKING POLICY GOVERNANCE_DB.POLICIES.PARTIAL_MASK_LAST4
  AS (val VARCHAR) RETURNS VARCHAR ->
  CASE
    WHEN IS_ROLE_IN_SESSION('GOVERNANCE_ADMIN') THEN val
    WHEN IS_ROLE_IN_SESSION('<EXEMPT_ROLE>') THEN val
    WHEN val IS NULL THEN NULL
    WHEN LENGTH(val) <= 4 THEN '****'
    ELSE CONCAT(REPEAT('*', LENGTH(val) - 4), RIGHT(val, 4))
  END
  COMMENT = 'Show last 4 chars only | Exempt: GOVERNANCE_ADMIN, <EXEMPT_ROLE>';
```

### Pattern: SHA-256 Hash (for analytics on masked data)

```sql
CREATE OR REPLACE MASKING POLICY GOVERNANCE_DB.POLICIES.HASH_MASK
  AS (val VARCHAR) RETURNS VARCHAR ->
  CASE
    WHEN IS_ROLE_IN_SESSION('GOVERNANCE_ADMIN') THEN val
    WHEN IS_ROLE_IN_SESSION('<EXEMPT_ROLE>') THEN val
    WHEN val IS NULL THEN NULL
    ELSE SHA2(val, 256)
  END
  COMMENT = 'SHA-256 hash mask for joinable anonymization | Exempt: GOVERNANCE_ADMIN, <EXEMPT_ROLE>';
```

### Pattern: Number Mask (for numeric PII)

```sql
CREATE OR REPLACE MASKING POLICY GOVERNANCE_DB.POLICIES.FULL_MASK_NUMBER
  AS (val NUMBER) RETURNS NUMBER ->
  CASE
    WHEN IS_ROLE_IN_SESSION('GOVERNANCE_ADMIN') THEN val
    WHEN IS_ROLE_IN_SESSION('<EXEMPT_ROLE>') THEN val
    WHEN val IS NULL THEN NULL
    ELSE -1
  END
  COMMENT = 'Numeric mask returns -1 | Exempt: GOVERNANCE_ADMIN, <EXEMPT_ROLE>';
```

### Pattern: Date Mask (for DOB / sensitive dates)

```sql
CREATE OR REPLACE MASKING POLICY GOVERNANCE_DB.POLICIES.DATE_MASK
  AS (val DATE) RETURNS DATE ->
  CASE
    WHEN IS_ROLE_IN_SESSION('GOVERNANCE_ADMIN') THEN val
    WHEN IS_ROLE_IN_SESSION('<EXEMPT_ROLE>') THEN val
    WHEN val IS NULL THEN NULL
    ELSE DATE_FROM_PARTS(1900, 1, 1)
  END
  COMMENT = 'Date mask returns 1900-01-01 | Exempt: GOVERNANCE_ADMIN, <EXEMPT_ROLE>';
```

### Pattern: Timestamp Mask

```sql
CREATE OR REPLACE MASKING POLICY GOVERNANCE_DB.POLICIES.TIMESTAMP_MASK
  AS (val TIMESTAMP_NTZ) RETURNS TIMESTAMP_NTZ ->
  CASE
    WHEN IS_ROLE_IN_SESSION('GOVERNANCE_ADMIN') THEN val
    WHEN IS_ROLE_IN_SESSION('<EXEMPT_ROLE>') THEN val
    WHEN val IS NULL THEN NULL
    ELSE '1900-01-01 00:00:00'::TIMESTAMP_NTZ
  END
  COMMENT = 'Timestamp mask | Exempt: GOVERNANCE_ADMIN, <EXEMPT_ROLE>';
```

## Tag-Based Masking Patterns

Tag-based masking automatically applies a masking policy to ANY
column that has a specific tag value. This is the RECOMMENDED
approach for scalable governance.

### Pattern: Bind Masking Policy to Tag Value

```sql
ALTER TAG GOVERNANCE_DB.POLICIES.PII_TYPE
  SET MASKING POLICY GOVERNANCE_DB.POLICIES.EMAIL_MASK
  FORCE;
```

IMPORTANT: One tag can only have ONE masking policy per data type.
If you need different masks for different PII types, use the
tag VALUE in a conditional policy:

### Pattern: Conditional Tag-Based Masking (Recommended)

```sql
CREATE OR REPLACE MASKING POLICY GOVERNANCE_DB.POLICIES.PII_TAG_MASK_STRING
  AS (val VARCHAR) RETURNS VARCHAR ->
  CASE
    WHEN IS_ROLE_IN_SESSION('GOVERNANCE_ADMIN') THEN val
    WHEN IS_ROLE_IN_SESSION('<EXEMPT_ROLE>') THEN val
    WHEN val IS NULL THEN NULL
    WHEN SYSTEM$GET_TAG_ON_CURRENT_COLUMN('GOVERNANCE_DB.POLICIES.PII_TYPE') = 'EMAIL'
      THEN CONCAT('*****@', SPLIT_PART(val, '@', 2))
    WHEN SYSTEM$GET_TAG_ON_CURRENT_COLUMN('GOVERNANCE_DB.POLICIES.PII_TYPE') = 'SSN'
      THEN CONCAT('***-**-', RIGHT(val, 4))
    WHEN SYSTEM$GET_TAG_ON_CURRENT_COLUMN('GOVERNANCE_DB.POLICIES.PII_TYPE') = 'PHONE'
      THEN CONCAT('***-***-', RIGHT(val, 4))
    WHEN SYSTEM$GET_TAG_ON_CURRENT_COLUMN('GOVERNANCE_DB.POLICIES.PII_TYPE') = 'CREDIT_CARD'
      THEN CONCAT(REPEAT('*', LENGTH(val) - 4), RIGHT(val, 4))
    WHEN SYSTEM$GET_TAG_ON_CURRENT_COLUMN('GOVERNANCE_DB.POLICIES.PII_TYPE') = 'NAME'
      THEN '********'
    WHEN SYSTEM$GET_TAG_ON_CURRENT_COLUMN('GOVERNANCE_DB.POLICIES.PII_TYPE') = 'ADDRESS'
      THEN '********'
    WHEN SYSTEM$GET_TAG_ON_CURRENT_COLUMN('GOVERNANCE_DB.POLICIES.PII_TYPE') = 'IP_ADDRESS'
      THEN '0.0.0.0'
    ELSE '********'
  END
  COMMENT = 'Conditional PII mask based on tag value | Exempt: GOVERNANCE_ADMIN, <EXEMPT_ROLE>';

ALTER TAG GOVERNANCE_DB.POLICIES.PII_TYPE
  SET MASKING POLICY GOVERNANCE_DB.POLICIES.PII_TAG_MASK_STRING
  FORCE;
```

Now ANY column tagged with PII_TYPE will auto-mask based on its
tag value. New columns tagged later are automatically protected.

## Row Access Policy Patterns

### Guardrails for Row Access Policies

- ALWAYS create a mapping table that maps roles to allowed
  attribute values — never hardcode role names in the policy
- ALWAYS include a governance admin bypass
- NEVER apply RAP and masking policy that reference the SAME column
- ONE row access policy per table (cannot stack multiple)
- Test with EXPLAIN to verify predicate pushdown
- Use IS_ROLE_IN_SESSION() for role hierarchy support

### Pattern: Role-Based Row Access Policy

```sql
CREATE TABLE IF NOT EXISTS GOVERNANCE_DB.POLICIES.ROLE_REGION_MAP (
  ROLE_NAME VARCHAR,
  ALLOWED_REGION VARCHAR
) COMMENT = 'Maps roles to allowed regions for row-level security';

INSERT INTO GOVERNANCE_DB.POLICIES.ROLE_REGION_MAP VALUES
  ('<ROLE_1>', '<REGION_1>'),
  ('<ROLE_2>', '<REGION_2>');

CREATE OR REPLACE ROW ACCESS POLICY GOVERNANCE_DB.POLICIES.REGION_RAP
  AS (region_col VARCHAR) RETURNS BOOLEAN ->
  IS_ROLE_IN_SESSION('GOVERNANCE_ADMIN')
  OR EXISTS (
    SELECT 1 FROM GOVERNANCE_DB.POLICIES.ROLE_REGION_MAP m
    WHERE IS_ROLE_IN_SESSION(m.ROLE_NAME)
      AND m.ALLOWED_REGION = region_col
  )
  COMMENT = 'Row-level filter by region | Admin bypass: GOVERNANCE_ADMIN';

ALTER TABLE <DB>.<SCHEMA>.<TABLE>
  ADD ROW ACCESS POLICY GOVERNANCE_DB.POLICIES.REGION_RAP
  ON (REGION);
```

### Pattern: Department-Based Row Access Policy

```sql
CREATE TABLE IF NOT EXISTS GOVERNANCE_DB.POLICIES.ROLE_DEPT_MAP (
  ROLE_NAME VARCHAR,
  ALLOWED_DEPT VARCHAR
);

CREATE OR REPLACE ROW ACCESS POLICY GOVERNANCE_DB.POLICIES.DEPT_RAP
  AS (dept_col VARCHAR) RETURNS BOOLEAN ->
  IS_ROLE_IN_SESSION('GOVERNANCE_ADMIN')
  OR EXISTS (
    SELECT 1 FROM GOVERNANCE_DB.POLICIES.ROLE_DEPT_MAP m
    WHERE IS_ROLE_IN_SESSION(m.ROLE_NAME)
      AND m.ALLOWED_DEPT = dept_col
  );
```

## Governance Audit Views

### Pattern: All Masked Columns Audit

```sql
CREATE OR REPLACE VIEW GOVERNANCE_DB.POLICIES.MASKED_COLUMNS_AUDIT AS
SELECT
  POLICY_DB || '.' || POLICY_SCHEMA || '.' || POLICY_NAME AS POLICY_FQN,
  REF_DATABASE_NAME || '.' || REF_SCHEMA_NAME || '.' || REF_ENTITY_NAME AS TABLE_FQN,
  REF_COLUMN_NAME AS COLUMN_NAME,
  POLICY_KIND
FROM SNOWFLAKE.ACCOUNT_USAGE.POLICY_REFERENCES
WHERE POLICY_KIND = 'MASKING_POLICY'
  AND DELETED IS NULL
ORDER BY TABLE_FQN, COLUMN_NAME;
```

### Pattern: All Row Access Policies Audit

```sql
CREATE OR REPLACE VIEW GOVERNANCE_DB.POLICIES.RAP_AUDIT AS
SELECT
  POLICY_DB || '.' || POLICY_SCHEMA || '.' || POLICY_NAME AS POLICY_FQN,
  REF_DATABASE_NAME || '.' || REF_SCHEMA_NAME || '.' || REF_ENTITY_NAME AS TABLE_FQN,
  REF_COLUMN_NAME AS RAP_COLUMN,
  POLICY_KIND
FROM SNOWFLAKE.ACCOUNT_USAGE.POLICY_REFERENCES
WHERE POLICY_KIND = 'ROW_ACCESS_POLICY'
  AND DELETED IS NULL
ORDER BY TABLE_FQN;
```

### Pattern: Tagged Columns Audit

```sql
SELECT
  TAG_DATABASE || '.' || TAG_SCHEMA || '.' || TAG_NAME AS TAG_FQN,
  TAG_VALUE,
  OBJECT_DATABASE || '.' || OBJECT_SCHEMA || '.' || OBJECT_NAME AS TABLE_FQN,
  COLUMN_NAME
FROM SNOWFLAKE.ACCOUNT_USAGE.TAG_REFERENCES
WHERE TAG_NAME = 'PII_TYPE'
  AND COLUMN_NAME IS NOT NULL
  AND DELETED IS NULL
ORDER BY TABLE_FQN, COLUMN_NAME;
```

## Deployment Order

1. **USE ROLE SYSADMIN** — create governance database + schema
2. **Tags** — CREATE TAG for PII_TYPE, SENSITIVITY, COMPLIANCE
3. **USE ROLE SYSADMIN** — create masking policies in governance schema
4. **Masking policies** — full, email, partial, hash, date, number, timestamp
5. **Conditional tag-based masking policy** — one per data type
6. **Row access policies** — with mapping tables
7. **Bind masking policies to tags** — ALTER TAG SET MASKING POLICY
8. **USE ROLE ACCOUNTADMIN** — for classification + policy assignment
9. **Run SYSTEM$CLASSIFY** — scan target schemas
10. **Apply tags** — to classified columns
11. **Apply row access policies** — to target tables
12. **Create audit views** — masked columns + RAP + tagged columns
13. **Set up Classification Profile** — for auto-detection (optional)
14. **Validation queries** — confirm everything works
15. **Test with non-exempt role** — verify masking is active

## RUN Protocol — Bulk Execution

Same batched approach as day0-foundation:

1. **Generate .sql file** first as documentation/fallback
2. **Execute in 4 batched role blocks**:
   - SYSADMIN: governance DB, schema, tags, masking policies, RAPs
   - ACCOUNTADMIN: classification, tag application, policy binding
   - Validation: audit queries
   - Test: SELECT as non-exempt role
3. **Between classification and tagging**, present results to user
   for confirmation (STOPPING POINT)

## Validation Queries

```sql
-- Confirm tags exist
SHOW TAGS IN SCHEMA GOVERNANCE_DB.POLICIES;

-- Confirm masking policies exist
SHOW MASKING POLICIES IN SCHEMA GOVERNANCE_DB.POLICIES;

-- Confirm row access policies exist
SHOW ROW ACCESS POLICIES IN SCHEMA GOVERNANCE_DB.POLICIES;

-- Confirm tag-based masking bindings
SELECT *
FROM TABLE(GOVERNANCE_DB.INFORMATION_SCHEMA.POLICY_REFERENCES(
  REF_ENTITY_NAME => 'GOVERNANCE_DB.POLICIES.PII_TYPE',
  REF_ENTITY_DOMAIN => 'TAG'));

-- Confirm tagged columns
SELECT TAG_VALUE, OBJECT_NAME, COLUMN_NAME
FROM SNOWFLAKE.ACCOUNT_USAGE.TAG_REFERENCES
WHERE TAG_NAME = 'PII_TYPE'
  AND DELETED IS NULL
ORDER BY TAG_VALUE;

-- Test masking with non-exempt role
USE ROLE <NON_EXEMPT_ROLE>;
SELECT <MASKED_COLUMN> FROM <DB>.<SCHEMA>.<TABLE> LIMIT 5;
```

## Anti-Patterns to Flag

| Anti-Pattern | Correct Approach |
|---|---|
| Using CURRENT_ROLE() in policies | Use IS_ROLE_IN_SESSION() for hierarchy support |
| One masking policy per column | One policy per data type, bind via tags |
| Masking primary/join keys | Never mask keys — breaks queries and joins |
| Hardcoding role names in RAP body | Use a mapping table for maintainability |
| Applying masking + RAP on same column | Use on different columns to avoid recursion |
| Creating policies in data schemas | Always use a dedicated governance schema |
| Skipping NULL handling in policies | Always return NULL for NULL input |
| Using CREATE OR REPLACE in production | Use IF NOT EXISTS after initial deployment |
| Not testing with non-exempt role | Always verify masking is active |
| Mixing data types in policy return | Input and output types must match exactly |

## Compliance Quick Reference

| Framework | Key Data Types | Recommended Protection |
|---|---|---|
| GDPR | Name, email, phone, address, DOB, IP | Tag-based masking + right-to-erasure process |
| CCPA | Name, email, SSN, financial, geolocation | Tag-based masking + consumer request workflow |
| HIPAA | MRN, DOB, SSN, diagnosis, medication | Tag-based masking + row access by care team |
| PCI-DSS | Credit card (PAN), CVV, cardholder name | Partial mask (last 4) + hash for analytics |

# Examples

## Example 1: Classify and Mask a Schema

**User:** Scan EDW_PRD_DB.ANALYTICS for PII. Mask any sensitive
columns. Only DATA_ENGINEER should see unmasked data.

**Expected output:** Run SYSTEM$CLASSIFY, present findings,
create tags + conditional masking policy, apply tags to
classified columns, validate with ANALYST role.

## Example 2: HIPAA Compliance on Patient Data

**User:** We have a PATIENTS table with MRN, name, DOB, SSN,
diagnosis. Only CARE_TEAM role should see full data. Other
roles should see masked values. Diagnosis should be visible
to ANALYST but patient identity should not.

**Expected output:** Tag-based masking on MRN/name/DOB/SSN
(RESTRICTED), diagnosis visible to all, row access policy
optional if multi-facility.

## Example 3: Row-Level Security by Region

**User:** Our SALES table has a REGION column. US_SALES role
should only see US rows. EU_SALES should only see EU rows.
GLOBAL_ANALYST sees everything.

**Expected output:** Role-region mapping table, row access
policy, GLOBAL_ANALYST as exempt, validation per role.

## Example 4: Full Governance Stack with Auto-Classification

**User:** Set up complete governance for a new environment.
Auto-classify new tables, tag-based masking, audit views,
Classification Profile on all schemas. Compliance: GDPR.

**Expected output:** Governance DB, all tags, all masking
policies, Classification Profile on every schema, conditional
tag-based masking, audit views, validation suite.
