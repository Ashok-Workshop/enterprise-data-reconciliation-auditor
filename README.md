# Automated Enterprise Data Reconciliation & Migration Auditor

> An AI agent skill for automated, forensic-grade verification of cloud and lakehouse database migrations.

## 📌 Problem Statement
Enterprise data migrations from legacy relational databases (SQL Server, Oracle) to modern lakehouse platforms (Microsoft Fabric, Databricks Delta Lake, Snowflake) often pass binary row-count validations while silently losing rows, dropping keys, or corrupting joins due to string padding and type-coercion mismatches. Manual forensic audits require dozens of engineering hours.

This skill automates the role of a Principal Data Quality Engineer by ingesting source/target telemetry, identifying schema/volume anomalies, diagnosing root causes, and issuing a deterministic **GO / NO-GO** sign-off report with production-ready remediation code.

## ⚙️ Architecture & Logic
1. **Schema & Collation Drift:** Flags precision loss (e.g., `DECIMAL(38,10)` to `DECIMAL(18,2)`), truncation, and nullability mismatches.
2. **Volumetric & Statistical Parity:** Evaluates gross row count variances, primary key cardinality, and null density shifts.
3. **Forensic Engine:** Identifies type-casting failures, trailing space issues, and partition skew.
4. **Remediation Plan:** Generates executable PySpark/SQL scripts for immediate remediation.

## 📁 Repository Structure
```text
├── SKILL.md                          # Full system instructions & prompt logic
├── README.md                         # Project documentation
└── examples/
    ├── real_world_case_study.md      # End-to-end 92M-row enterprise case study
    └── sample_input.json             # Standard input payload for the skill