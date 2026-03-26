# Day-0 Platform Setup App — Complete Architecture & Documentation

---

## Table of Contents

1. [What is Day-0 Platform Setup?](#1-what-is-day-0-platform-setup)
2. [Problem Statement](#2-problem-statement)
3. [Solution Overview](#3-solution-overview)
4. [Architecture Diagram](#4-architecture-diagram)
5. [Technology Stack](#5-technology-stack)
6. [Detailed Tab-by-Tab Flow](#6-detailed-tab-by-tab-flow)
7. [Skills & Cortex Complete Integration](#7-skills--cortex-complete-integration)
8. [Security & Governance](#8-security--governance)
9. [State Management](#9-state-management)
10. [Key Functions Explained](#10-key-functions-explained)
11. [Demo Walkthrough (3-Minute Script)](#11-demo-walkthrough-3-minute-script)

---

## 1. What is Day-0 Platform Setup?

**Day-0** refers to the very first day of setting up a data platform — before any data lands, before any pipelines run, before any analysts write queries. It is the foundation layer that everything else is built upon.

In a Snowflake environment, Day-0 setup typically involves creating:

- **Databases** — logical containers for your data
- **Schemas** — organizational units within databases (e.g., RAW, STAGING, ANALYTICS)
- **Warehouses** — compute resources for running queries
- **Roles** — access control identities (e.g., DATA_ENGINEER, ANALYST, ADMIN)
- **Grants & Privileges** — permissions connecting roles to objects
- **Tables** — initial table structures (e.g., an ORDERS table for the sales team)
- **Masking Policies** — data protection rules for sensitive/PII columns
- **Resource Monitors** — cost control guardrails

Getting this foundation wrong means every team, pipeline, and dashboard built on top inherits that technical debt.

---

## 2. Problem Statement

| Challenge | Impact |
|-----------|--------|
| **Manual SQL scripting** | Engineers hand-write hundreds of lines of DDL, grants, and config — error-prone and slow |
| **No standardization** | Every engineer sets things up differently — naming conventions, role hierarchies, and grant patterns vary |
| **Security gaps** | Roles are often overprivileged, masking policies are forgotten, and no audit trail exists |
| **No validation** | Objects are created but never verified — missing grants or schemas are discovered weeks later |
| **Tribal knowledge** | Only senior engineers know the "right" way to set up a platform — this knowledge isn't codified |
| **Time cost** | A full platform setup typically takes **2-5 days** of manual work |

---

## 3. Solution Overview

The **Day-0 Platform Setup App** is a Streamlit-in-Snowflake (SiS) application that provides a **guided, AI-assisted, governed workflow** for setting up a complete Snowflake data platform.

### Core Principles

1. **AI-Assisted Generation** — Snowflake Cortex Complete (LLM) generates all SQL based on user inputs
2. **Human-in-the-Loop** — No SQL executes without explicit human review and approval
3. **Governed by Design** — Audit logging, approval gates, and validation are built-in
4. **Skill-Based Architecture** — Domain expertise is codified as reusable templates (skills)
5. **End-to-End Validation** — Post-deployment verification confirms every object exists

### What It Replaces

```
BEFORE: Engineer opens a blank SQL worksheet, writes 200+ lines from memory, runs them
        one-by-one, hopes nothing is missing, moves on.

AFTER:  Engineer opens the app, selects a template, fills in a form, reviews AI-generated
        SQL, clicks deploy, and gets a validated audit report — in minutes.
```

---

## 4. Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                        DAY-0 PLATFORM SETUP APP - ARCHITECTURE                  │
└─────────────────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────────────────┐
│                              STREAMLIT IN SNOWFLAKE (SiS)                        │
│                                                                                  │
│  ┌─────────────┐   ┌─────────────┐   ┌──────────┐   ┌────────┐   ┌──────────┐  │
│  │  Tab 1:     │   │  Tab 2:     │   │  Tab 3:  │   │ Tab 4: │   │  Tab 5:  │  │
│  │  Configure  │──>│  Generate   │──>│  Review  │──>│ Deploy │──>│ Summary  │  │
│  │  & Input    │   │  SQL        │   │ & Approve│   │        │   │ Validate │  │
│  └──────┬──────┘   └──────┬──────┘   └────┬─────┘   └───┬────┘   └────┬─────┘  │
│         │                 │               │              │              │         │
└─────────┼─────────────────┼───────────────┼──────────────┼──────────────┼─────────┘
          │                 │               │              │              │
          ▼                 ▼               ▼              ▼              ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                           st.session_state (State Management)                    │
│                                                                                  │
│  ┌──────────────┐ ┌──────────────┐ ┌───────────────┐ ┌────────────────────────┐ │
│  │ step         │ │ generated_sql│ │review_confirmed│ │ deployment_result      │ │
│  │ selected_skill│ │ validation_sql│ │               │ │ exec_log               │ │
│  │ selected_model│ │              │ │               │ │ validation_results     │ │
│  └──────────────┘ └──────────────┘ └───────────────┘ └────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────────────┘

========================== DETAILED TAB FLOW ======================================

TAB 1: CONFIGURE & INPUT
┌──────────────────────────────┐
│  User Inputs:                │
│  ├─ Select Skill/Template    │──── Skills: Full Platform, RBAC, Warehouse...
│  ├─ Select LLM Model        │──── llama3.1-70b, mistral-large, etc.
│  ├─ Step 1: Basic Info       │──── DB name, schemas, environment
│  ├─ Step 2: Warehouse Config │──── Sizes, auto-suspend, scaling
│  └─ Step 3: Roles & Access  │──── Role hierarchy, grants, users
└──────────────┬───────────────┘
               │
               ▼
TAB 2: GENERATE SQL
┌──────────────────────────────┐       ┌───────────────────────────────────┐
│  Prompt Construction         │       │     SNOWFLAKE CORTEX COMPLETE     │
│  ├─ Skill template           │       │                                   │
│  ├─ + User parameters        │──────>│  snowflake.cortex.Complete(       │
│  ├─ + Best practices rules   │       │    model = selected_model,        │
│  └─ = Structured prompt      │       │    prompt = assembled_prompt      │
│                              │<──────│  )                                │
│  Output:                     │       │                                   │
│  ├─ generated_sql            │       │  Runs NATIVELY inside Snowflake   │
│  └─ validation_sql           │       │  No external API / No data leaves │
└──────────────┬───────────────┘       └───────────────────────────────────┘
               │
               ▼
TAB 3: REVIEW & APPROVE
┌──────────────────────────────┐
│  parse_sql_statements()      │──── Splits multi-statement SQL
│  categorize_statement()      │──── DDL / Grant / Warehouse / Role / Other
│  get_stmt_summary()          │──── Human-readable summary per stmt
│                              │
│  ┌────────────────────────┐  │
│  │ [DDL] Stmt 1: CREATE  │  │──── Expandable per statement
│  │ [Grant] Stmt 2: GRANT │  │
│  │ [Warehouse] Stmt 3:.. │  │
│  └────────────────────────┘  │
│                              │
│  ⚠ "I have reviewed and     │──── HUMAN-IN-THE-LOOP GATE
│     approve this SQL"        │     (Nothing deploys without approval)
└──────────────┬───────────────┘
               │ review_confirmed = True
               ▼
TAB 4: DEPLOY
┌──────────────────────────────┐       ┌───────────────────────────────────┐
│  Deployment Plan             │       │        SNOWFLAKE ENGINE           │
│  ├─ N statements total       │       │                                   │
│  ├─ Breakdown by category    │       │  ┌─────────────────────────────┐  │
│  │                           │       │  │ CREATE DATABASE ...         │  │
│  │  execute_sql_scripts()    │──────>│  │ CREATE SCHEMA ...           │  │
│  │  ├─ Sequential execution  │       │  │ CREATE WAREHOUSE ...        │  │
│  │  ├─ Per-stmt timing       │       │  │ CREATE ROLE ...             │  │
│  │  ├─ Error capture         │       │  │ GRANT ROLE ...              │  │
│  │  └─ Category tagging      │<──────│  │ CREATE MASKING POLICY ...   │  │
│  │                           │       │  │ ALTER TABLE SET MASKING ... │  │
│  │  log_audit()              │       │  └─────────────────────────────┘  │
│  └─ Audit trail recorded     │       │                                   │
└──────────────┬───────────────┘       └───────────────────────────────────┘
               │
               ▼
TAB 5: SUMMARY & VALIDATE
┌──────────────────────────────┐       ┌───────────────────────────────────┐
│  Metrics Dashboard           │       │     VALIDATION ENGINE             │
│  ├─ Total / Success / Fail   │       │                                   │
│  ├─ Duration                 │       │  run_validation()                 │
│  ├─ Category breakdown       │       │  ├─ SHOW DATABASES LIKE ...      │
│  │                           │       │  ├─ SHOW SCHEMAS IN ...          │
│  │  Validation               │──────>│  ├─ SHOW WAREHOUSES LIKE ...     │
│  │  ├─ Found: ✓              │<──────│  ├─ SHOW ROLES LIKE ...          │
│  │  ├─ Missing: ✗            │       │  └─ SHOW TABLES LIKE ...         │
│  │  ├─ Errors: ⚠             │       │                                   │
│  │                           │       └───────────────────────────────────┘
│  │  Execution Log            │
│  │  └─ Per-stmt detail       │
│  │                           │
│  │  Copy Execution Report    │──── Exportable audit report
└──────────────────────────────┘
```

---

## 5. Technology Stack

### Streamlit in Snowflake (SiS)

**What it is:** Streamlit is a Python-based framework for building interactive web applications. Snowflake hosts Streamlit natively — meaning the app runs **inside** your Snowflake account, not on an external server.

**Why it matters:**
- No separate infrastructure to manage
- The app inherits the user's Snowflake session, role, and permissions
- Data never leaves the Snowflake security perimeter
- Deployed and accessed directly from the Snowflake UI (Snowsight)

**How it's used in this app:** The entire UI — tabs, forms, buttons, metrics, expanders — is built with Streamlit components like `st.tabs()`, `st.button()`, `st.metric()`, `st.expander()`, and `st.code()`.

### Snowflake Cortex Complete

**What it is:** A Snowflake-native function that provides access to large language models (LLMs) like Llama, Mistral, and others — directly via SQL or Python, without any external API calls.

**Why it matters:**
- No API keys to manage
- No data leaves your Snowflake account (critical for compliance)
- Billed through Snowflake credits (no separate LLM vendor billing)
- Supports multiple models — users can choose based on speed vs. quality tradeoff

**How it's used in this app:** When the user clicks "Generate SQL," the app:
1. Assembles a structured prompt from the skill template + user parameters
2. Calls `snowflake.cortex.Complete(model, prompt)`
3. Receives generated SQL back from the LLM
4. Stores it in `st.session_state.generated_sql`

```python
# Simplified example of the Cortex Complete call
from snowflake.cortex import Complete

prompt = skill_template.format(
    database_name=user_input_db,
    schemas=user_input_schemas,
    warehouse_size=user_input_wh_size,
    roles=user_input_roles
)

generated_sql = Complete(model="llama3.1-70b", prompt=prompt)
```

### Snowflake SQL Engine

**What it is:** The core Snowflake execution engine that processes SQL statements.

**How it's used in this app:** The `execute_sql_scripts()` function runs each generated SQL statement sequentially against Snowflake, capturing success/failure status, timing, and error messages for each statement.

---

## 6. Detailed Tab-by-Tab Flow

### Tab 1: Configure & Input

**Purpose:** Gather all the information needed to generate the platform setup SQL.

**How it works:**

1. **Skill Selection** — The user picks a skill/template from a dropdown (e.g., "Full Platform Setup," "RBAC Configuration," "Warehouse Provisioning"). Each skill defines what inputs are needed and what SQL patterns to generate.

2. **Model Selection** — The user picks which Cortex LLM to use. Different models offer different tradeoffs:
   - `llama3.1-70b` — High quality, slightly slower
   - `mistral-large` — Fast, good for simpler setups
   - Other available Cortex models

3. **Step-by-Step Input** — The form is broken into logical steps:
   - **Step 1: Basic Info** — Database name, list of schemas, environment (dev/staging/prod)
   - **Step 2: Warehouse Config** — Warehouse sizes, auto-suspend timeout, scaling policies
   - **Step 3: Roles & Access** — Role names, hierarchy, which roles get access to what

4. **State Tracking** — Each input is stored in `st.session_state` so it persists across reruns. The `step` variable tracks which step the user is on.

### Tab 2: Generate SQL

**Purpose:** Take the user's inputs, construct a prompt, call Cortex Complete, and produce deployment-ready SQL.

**How it works:**

1. **Prompt Assembly** — The app takes the selected skill's template and injects the user's parameters into it. The prompt includes:
   - The specific objects to create (databases, schemas, etc.)
   - Naming conventions and best practices
   - Instructions to generate validation SQL alongside the deployment SQL

2. **Cortex Complete Call** — The assembled prompt is sent to the selected LLM via `snowflake.cortex.Complete()`. The LLM returns structured SQL.

3. **Output Parsing** — The returned SQL is split into:
   - `generated_sql` — The main deployment SQL (CREATE, GRANT, ALTER statements)
   - `validation_sql` — Verification queries (SHOW, DESCRIBE statements) to run post-deploy

4. **Display** — The generated SQL is shown to the user in a code block, ready for review.

### Tab 3: Review & Approve

**Purpose:** Provide a human-in-the-loop review gate so no SQL executes without explicit approval.

**How it works:**

1. **Statement Parsing** — `parse_sql_statements()` splits the generated SQL into individual statements by semicolons, handling edge cases like strings containing semicolons.

2. **Categorization** — `categorize_statement()` classifies each statement:
   - **DDL** — CREATE DATABASE, CREATE SCHEMA, CREATE TABLE
   - **Grant** — GRANT ROLE, GRANT USAGE, GRANT SELECT
   - **Warehouse** — CREATE WAREHOUSE, ALTER WAREHOUSE
   - **Role** — CREATE ROLE, ALTER ROLE
   - **Policy** — CREATE MASKING POLICY, ALTER TABLE SET MASKING POLICY
   - **Other** — Anything else

3. **Summary Generation** — `get_stmt_summary()` produces a human-readable one-line summary for each statement (e.g., "Create database SALES_DB").

4. **Expandable Review** — Each statement is displayed in a collapsible expander showing the category tag, summary, and full SQL code. This lets reviewers quickly scan categories or drill into specific statements.

5. **Approval Gate** — The user must click **"I have reviewed and approve this SQL"** to proceed. This sets `review_confirmed = True`. Without this, the Deploy tab is locked.

6. **Reset Option** — At any point, the user can click **"Reset & Restart"** to clear all state and start over.

### Tab 4: Deploy

**Purpose:** Execute the approved SQL against Snowflake with real-time progress tracking.

**How it works:**

1. **Deployment Plan** — Before executing, the app shows a summary:
   - Total number of statements
   - Breakdown by category (e.g., "DDL: 5, Grant: 8, Warehouse: 2")

2. **Pre-flight Check** — The app verifies `review_confirmed == True`. If not, it blocks deployment with a warning.

3. **Execution** — When the user clicks **"Deploy to Snowflake"**:
   - A deployment ID is generated (e.g., `DEP_20260326_143022`)
   - `execute_sql_scripts()` runs each statement sequentially
   - For each statement, it records:
     - Status (OK or FAIL)
     - Category
     - Execution time
     - Error message (if failed)
   - Results are stored in `exec_log` (per-statement) and `deployment_result` (summary)

4. **Result Display** — Immediately after execution:
   - **SUCCESS** — Green success box with count and duration
   - **PARTIAL** — Warning with success/fail counts
   - **FAILED** — Error message with details

5. **Error Handling** — Failed statements are captured but don't stop execution of subsequent statements. Errors are shown in expandable sections.

6. **Audit Logging** — `log_audit()` records the deployment event including skill used, model used, and final status.

### Tab 5: Summary & Validate

**Purpose:** Post-deployment dashboard with metrics, validation, and exportable audit reports.

**How it works:**

1. **Metrics Dashboard** — Four key metrics displayed as cards:
   - Total Statements
   - Successful
   - Failed
   - Duration (seconds)

2. **Category Breakdown** — Shows success rate per category (e.g., "DDL: 5/5, 100% success", "Grant: 7/8, 88% success").

3. **Validation Engine** — `run_validation()` executes the validation SQL generated in Tab 2. For each object:
   - **FOUND** — Object exists in Snowflake (green checkmark)
   - **MISSING** — Object was not created (red X)
   - **ERROR** — Validation query itself failed (warning icon)

4. **Execution Log** — Detailed per-statement log showing:
   - Status icon (checkmark or X)
   - Category tag
   - Statement number
   - Summary
   - Error text (if failed)
   - Execution time

5. **Copy Execution Report** — Generates a text-based report that can be copied and shared, including deployment status, per-statement results, and validation results.

---

## 7. Skills & Cortex Complete Integration

### What Are Skills?

Skills are **pre-built, reusable prompt templates** that encode domain expertise. Think of them as "recipes" for different types of platform setup tasks.

Each skill defines:
- **What inputs it needs** (e.g., database name, schema list, role names)
- **What SQL patterns to generate** (e.g., CREATE DATABASE, GRANT USAGE)
- **What best practices to follow** (e.g., least-privilege access, naming conventions)
- **What validation to run** (e.g., SHOW DATABASES LIKE 'X')

### Example Skills

| Skill | What It Does |
|-------|-------------|
| **Full Platform Setup** | Creates databases, schemas, warehouses, roles, grants, and policies — complete end-to-end |
| **RBAC Configuration** | Focuses on role hierarchy, grants, and access control only |
| **Warehouse Provisioning** | Creates and configures warehouses with auto-suspend, scaling, and resource monitors |
| **Data Masking Setup** | Creates masking policies for PII columns and applies them to tables |
| **Table Creation** | Creates table structures with appropriate data types and constraints |

### How Cortex Complete Is Called

```
User Input  ──>  Skill Template  ──>  Assembled Prompt  ──>  Cortex Complete  ──>  SQL Output
```

1. The user fills in the form (Tab 1)
2. The app selects the matching skill template
3. User parameters are injected into the template to create a detailed prompt
4. The prompt is sent to `snowflake.cortex.Complete(model, prompt)`
5. The LLM returns structured, deployment-ready SQL
6. The SQL is parsed and displayed for review

### Why This Matters

- **Junior engineers** get the same quality output as senior platform architects
- **Consistency** — every setup follows the same patterns and best practices
- **Speed** — what took days now takes minutes
- **No data leakage** — Cortex Complete runs inside Snowflake; no external API calls

---

## 8. Security & Governance

### Approval Gate (Human-in-the-Loop)

The app enforces a strict approval flow:

```
Generate SQL  ──>  Review Every Statement  ──>  Explicit Approval Click  ──>  Deploy
                                                        │
                                              Without this click,
                                              the Deploy tab is LOCKED
```

This ensures no auto-generated SQL is blindly executed. A human must review and approve every statement.

### Audit Logging

Every deployment is logged with:
- **Timestamp** — when the deployment was executed
- **Skill used** — which template/skill was selected
- **Model used** — which LLM generated the SQL
- **Status** — SUCCESS, PARTIAL, or FAILED
- **Statement-level detail** — per-statement success/failure with error messages

### Data Masking Integration

The app can generate masking policies for sensitive columns:

| Column | Mask Type | What Unauthorized Users See |
|--------|-----------|----------------------------|
| `CUSTOMER_SSN` | Full mask | `***-**-****` |
| `CREDIT_CARD_NUMBER` | Partial mask | `****-****-****-1234` |
| `CUSTOMER_EMAIL` | Partial mask | `j***@****.com` |
| `CUSTOMER_PHONE` | Partial mask | `***-***-5678` |
| `BILLING_ADDRESS` | Full mask | `*****` |

These policies are created as part of the generated SQL and applied to tables via `ALTER TABLE ... SET MASKING POLICY`.

### Post-Deployment Validation

After deployment, the validation engine confirms every object exists:

```
✓ DATABASE SALES_DB — FOUND
✓ SCHEMA SALES_DB.RAW — FOUND
✓ SCHEMA SALES_DB.STAGING — FOUND
✓ WAREHOUSE SALES_WH — FOUND
✓ ROLE DATA_ENGINEER — FOUND
✓ TABLE SALES_DB.PUBLIC.ORDERS — FOUND
✗ MASKING POLICY SSN_MASK — MISSING    <-- immediate visibility into gaps
```

---

## 9. State Management

The app uses Streamlit's `st.session_state` to persist data across user interactions (button clicks, tab switches, reruns).

### Key State Variables

| Variable | Type | Purpose |
|----------|------|---------|
| `step` | int | Tracks which configuration step the user is on (1, 2, or 3) |
| `generated_sql` | str | The SQL generated by Cortex Complete |
| `validation_sql` | str | The validation queries to run post-deploy |
| `review_confirmed` | bool | Whether the user has approved the SQL |
| `deployment_complete` | bool | Whether deployment has been executed |
| `deployment_result` | dict | Summary: status, total, successful, failed, duration |
| `exec_log` | list | Per-statement execution log with status, category, timing, errors |
| `validation_results` | list | Per-object validation results (FOUND/MISSING/ERROR) |

### Reset Flow

When the user clicks "Reset & Restart," all state variables are cleared:

```python
st.session_state.generated_sql = None
st.session_state.deployment_complete = False
st.session_state.deployment_result = None
st.session_state.exec_log = []
st.session_state.review_confirmed = False
st.session_state.validation_sql = None
st.session_state.validation_results = []
st.session_state.step = 1
```

This returns the app to a clean state for a new deployment.

---

## 10. Key Functions Explained

### `parse_sql_statements(sql)`
**What it does:** Takes a multi-statement SQL string and splits it into individual statements.
**How:** Splits on semicolons while respecting string literals and comments.
**Returns:** A list of individual SQL statement strings.

### `categorize_statement(stmt)`
**What it does:** Classifies a SQL statement into a category.
**How:** Pattern-matches the first keywords of the statement (CREATE DATABASE → DDL, GRANT → Grant, CREATE WAREHOUSE → Warehouse, etc.).
**Returns:** A category string like "DDL", "Grant", "Warehouse", "Role", "Policy", or "Other".

### `get_stmt_summary(stmt)`
**What it does:** Generates a human-readable one-line summary.
**How:** Extracts the key action and object name from the statement.
**Returns:** A string like "Create database SALES_DB" or "Grant usage on schema RAW to role ANALYST".

### `execute_sql_scripts(sql, deployment_id)`
**What it does:** Executes all SQL statements sequentially against Snowflake.
**How:** Iterates through parsed statements, runs each via the Snowflake session, captures results.
**Returns:** A dict with `status` (SUCCESS/PARTIAL/FAILED), `total`, `successful`, `failed`, `duration`, and `errors`.

### `run_validation(validation_sql)`
**What it does:** Runs validation queries to confirm objects exist.
**How:** Executes SHOW/DESCRIBE queries and checks if objects are found.
**Returns:** A list of dicts with `object`, `status` (FOUND/MISSING/ERROR), and `detail`.

### `log_audit(skill, model, prompt, sql, reviewed, approved, status)`
**What it does:** Records an audit trail entry for the deployment.
**How:** Logs metadata about who deployed what, when, using which skill and model, and the outcome.

### `safe_rerun()`
**What it does:** Triggers a Streamlit rerun to refresh the UI after state changes.
**How:** Wraps `st.rerun()` (or `st.experimental_rerun()` for older versions) with error handling.

---

## 11. Demo Walkthrough (3-Minute Script)

### Opening (30 seconds)
> "Platform setup in Snowflake — databases, schemas, warehouses, roles, grants — typically means hundreds of lines of hand-written SQL, no standardization, and no validation. **This app fixes that.**"

### Show the Flow (2 minutes)

1. **Configure** (20s) — "Pick a template, choose an AI model, fill in your requirements. Guided form — no guesswork."

2. **Generate** (20s) — "One click — Cortex Complete generates all the SQL: DDL, grants, warehouses, roles. Categorized and structured."

3. **Review & Approve** (20s) — "Human-in-the-loop. Every statement is categorized and expandable. Nothing deploys without explicit approval. This is governance by design."

4. **Deploy** (30s) — "One click to execute. Real-time success/failure tracking per statement."

5. **Validate** (30s) — "Post-deploy verification — confirms every object actually exists in Snowflake. Full audit report you can export."

### Close (30 seconds)
> "**Before:** days of manual SQL, inconsistent setups, no validation.
> **After:** minutes, repeatable, governed, AI-assisted, fully validated.
> Day-0 done right — every single time."

---

*Built with Streamlit in Snowflake + Snowflake Cortex Complete*
