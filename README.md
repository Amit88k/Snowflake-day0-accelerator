# Snowflake-day0-accelerator
A 100% native Snowflake Streamlit app powered by  Cortex AI to automate Day-0 environment provisioning, governance, and RBAC from weeks to minutes. Built by Team Ex Nihilo


# 🚀 Day-0 Accelerator on Snowflake

*Built with Cortex Code | Delivered via Streamlit | Powered by Cortex AI | 100% Native Snowflake*

Setting up a Snowflake environment from scratch (Day-0) is traditionally slow, manual, and inconsistent. Typically, it requires 500+ DDL/DCL handwritten statements and 2-3 weeks of senior engineer time per project.

*Day-0 Accelerator* transforms this 15-day manual engineering sprint into a 15-minute guided conversation.

---

## 🎯 The Problem
Manual Day-0 setups consistently ship with critical gaps:
* *Inconsistent Naming:* Causes broken pipelines.
* *Incomplete RBAC:* Leads to over-privileged users.
* *Missing Masking:* Immediate compliance risk with exposed PII/PHI.
* *Warehouse Guesswork:* Runaway credits due to poor sizing.

## 💡 Our Solution
We engineered a solution that delivers a production-ready, governed Snowflake platform.

### Key Features
* *Unified Wizard:* A 4-step Streamlit UI that captures all architectural requirements.
* *Cortex AI Engine:* Generates production-grade, idempotent SQL automatically.
* *One-Click Governance:* Automated RBAC, dynamic data masking, and ingestion with built-in guardrails.
* *Approval Gate:* A mandatory 'Human-in-the-loop' review ensures no code executes without explicit confirmation.
* *Dependency-Aware Orchestration:* Executes platform setup in the correct order (e.g., Databases -> Schemas -> Roles -> Grants).
* *Automated Validation:* Post-deployment queries confirm object existence and function.

---

## 🏗️ Architecture & Build Design
The entire application was authored using *Cortex Code (CoCo)*.

1.  *The Streamlit App:* Generated through natural language python code.
2.  *The Skill Registry:* Modular AI prompt templates with embedded guardrails stored as Snowflake table rows, requiring zero application changes to scale.
3.  *Three Lines of Defense:* * *AI Guardrails:* Blocks destructive commands (DROP/DELETE).
    * *Human Approval Gate*.
    * *Validation Dashboard*.

---

## 📊 Business Impact
* *Speed:* Provisioning cut from weeks to < 30 minutes.
* *Standardization:* Zero drift between teams; zero manual SQL code.
* *Cost Optimization:* Auto-suspend, resource monitors, and credit quotas configured before the first query runs.
* *Compliance:* Audit log satisfies GDPR, CCPA, HIPAA, and PCI-DSS requirements.

---

## 🔮 Future Scope
* *Advanced Capabilities:* AI modules for storage optimization and pipeline monitoring.
* *DevOps Synergy:* Enabling "SQL-as-Code" by exporting AI-generated artifacts directly into GitHub/CI-CD workflows.
* *Enterprise Scale:* Global deployments across an entire Snowflake Organization.

---

## 👥 Team Ex Nihilo
* Amit Khandelwal
* Gaurav Bhatt
* Akshaya Jayakumar

Built for Cocothon by kipi.ai (A WNS Company)
