# ⚙️ Execution Flow: Snowflake Cortex AI Orchestration

This document outlines the core execution pipeline for the Day-0 Accelerator. The architecture strictly isolates application logic from AI prompt engineering, ensuring secure, predictable, and governed SQL generation natively within Snowflake.

## 1. Detailed System Flow Diagram

```text
=====================================================================================
                      PHASE 1: CONTEXT RETRIEVAL & ASSEMBLY
=====================================================================================

  [ Streamlit UI ]                                  [ Snowflake Database ]
         │                                                    │
         │ 1. User selects module (e.g., Compute)             │
         ▼                                                    ▼
  ( customer_config )                        [ SKILL_REGISTRY Table ]
  { domains, sizing,                         ( Rows of AI prompt templates )
    RBAC needs }                                              │
         │                                                    │ 2. load_skill("COMPUTE")
         │                                                    ▼
         │                                         ( skill_content )
         │                                         { full SKILL.md text containing:
         │                                           - Snowflake Best Practices
         │                                           - Strict SQL Guardrails }
         │                                                    │
=====================================================================================
                      PHASE 2: AI INFERENCE (CORTEX)
=====================================================================================
         │                                                    │
         │ 3a. Passed as prompt=                            │ 3b. Injected as system=
         │     (User Intent)                                  │     (Absolute Rules)
         ▼                                                    ▼
    ┌───────────────────────────────────────────────────────────────┐
    │ SNOWFLAKE.CORTEX.COMPLETE()                                   │
    │   • model  = "claude-opus-4-6"                                │
    │   • prompt = customer_config_as_prompt                        │
    │   • system = skill_content                                    │
    └───────────────────────────────────────────────────────────────┘
                                  │
                                  │ 4. LLM Generation
                                  ▼
                        [ Generated SQL ]
                ( Guaranteed to follow skill guardrails )

=====================================================================================
                      PHASE 3: HUMAN REVIEW & EXECUTION
=====================================================================================
                                  │
                                  │ 5. Render to UI
                                  ▼
                         [ Streamlit UI ]
                     st.code(Generated_SQL)
                                  │
                                  │ 6. Human-in-the-loop
                                  ▼
                         [ User Review ]
                  ( Visually verifies no DROP/DELETE )
                                  │
                                  │ 7. Explicit Approval (Click "Deploy")
                                  ▼
                       [ Snowflake Engine ]
                     session.sql().collect()
                                  │
                                  │ 8. Native Execution
                                  ▼
                    🚀 Infrastructure Deployed 🚀



⚙️ Infrastructure Deployment Workflow
This document details the three-phase pipeline used to translate user requirements into governed, executable Snowflake infrastructure code.

Phase 1: Context Retrieval & Assembly
User Inputs (customer_config): The workflow begins when a user finishes inputting architectural requirements into the Streamlit wizard (e.g., domain names, warehouse sizes, roles). This data is aggregated into a plain-text format (customer_config_as_prompt).

The SKILL_REGISTRY Table: Instead of hardcoding AI instructions in the Python application, the app queries a native Snowflake table called SKILL_REGISTRY.

load_skill("COMPUTE"): The app retrieves the specific skill_content row for the active module. This content acts as the master rulebook, containing strict instructions, expected formatting, and safety guardrails.

Phase 2: AI Inference via Cortex
The application orchestrates generation using CORTEX.COMPLETE(), utilizing a strict separation of concerns via the message array:

The system= parameter: The skill_content retrieved from the database is passed as the system prompt. This gives it the highest priority in the LLM's context window, forcing the AI to act as a strict Snowflake Architect.

The prompt= parameter: The customer_config is passed as the standard user prompt.

Execution: The designated model (e.g., Claude 3.5 Sonnet or Claude 3 Opus) processes the user intent through the lens of the system guardrails, ensuring the output is perfectly formatted, idempotent SQL.

Phase 3: Human Review & Execution (Three Lines of Defense)
st.code() Rendering: The generated SQL is passed back to the frontend and displayed using Streamlit's code block component for clear visibility.

Human-in-the-Loop: The application pauses execution. A human operator must visually verify the deployment plan, fulfilling a critical governance and security requirement to prevent accidental destructive actions.

session.sql().collect(): Once the user clicks "Approve/Deploy", the Streamlit app iterates through the validated SQL script and executes it directly against the active Snowflake session, provisioning the infrastructure safely and consistently.



## Architecture Mermaid diagram

graph TD
    %% Phase 1
    subgraph Phase_1 [PHASE 1: CONTEXT RETRIEVAL & ASSEMBLY]
        A[Streamlit Wizard] -->|User Inputs| B(customer_config)
        C[(SKILL_REGISTRY Table)] -->|load_skill| D{skill_content}
        style Phase_1 fill:#f9f9f9,stroke:#333,stroke-width:2px
    end

    %% Phase 2
    subgraph Phase_2 [PHASE 2: AI INFERENCE via CORTEX]
        B -->|prompt=| E[CORTEX.COMPLETE]
        D -->|system=| E
        E -->|Execution| F[Generated SQL]
        style Phase_2 fill:#e1f5fe,stroke:#01579b,stroke-width:2px
    end

    %% Phase 3
    subgraph Phase_3 [PHASE 3: HUMAN REVIEW & EXECUTION]
        F -->|Render| G[st.code Block]
        G -->|Visual Verify| H{Human-in-the-Loop}
        H -->|Approve/Deploy| I[session.sql.collect]
        I -->|Provision| J[🚀 Infrastructure Deployed]
        style Phase_3 fill:#e8f5e9,stroke:#2e7d32,stroke-width:2px
    end

    %% Flow
    Phase_1 --> Phase_2
    Phase_2 --> Phase_3
