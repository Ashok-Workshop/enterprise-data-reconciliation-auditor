# Real-World Case Study: 92-Million Row Healthcare Migration Audit

## Context & Scenario
A healthcare enterprise migrated 91.9 million records from an on-premises legacy SQL Server data warehouse to a Microsoft Fabric Delta Lakehouse (`gold.patient_mapping`). 
While the automated copy pipeline in Azure Data Factory reported an exit status of `Succeeded`, downstream reporting identified join anomalies and missing encounter records.

---

## Audit Execution & Skill Output

### 1. Migration Sign-off Verdict
- **Status:** **REJECTED (NO-GO)**
- **Integrity Score:** **96.8%**
- **Executive Summary:** Reconciled 91,900,083 source records against 91,865,894 target records. The migration dropped exactly 34,189 records during bronze-to-gold ingestion and introduced silent nullification in `ENCOUNTER_NUM`. Downstream joins will fail until remediation is applied.

### 2. Reconciliation Scorecard
| Metric Dimension | Source System (SQL Server) | Target System (Fabric Delta) | Variance (Abs / %) | Status |
|---|---|---|---|---|
| Total Row Count | 91,900,083 | 91,865,894 | -34,189 (-0.037%) | **FAIL** |
| Primary Key Cardinality | 66,132,496 | 66,132,496 | 0 (0.00%) | **PASS** |
| Schema Conformance | Strict types | Relaxed Delta types | `ACTIVE_FLAG` length dropped | **WARN** |
| Null Parity (`ENCOUNTER_NUM`)| 0 nulls | 34,189 nulls | +34,189 (+100%) | **FAIL** |

### 3. Detailed Drift & Forensic Findings

- **[CRITICAL] Silent Record Drop & Key Nullification on Join**
  - **Component / Column:** `ENCOUNTER_NUM`
  - **Observed Behavior:** Target table is short exactly 34,189 rows; 34,189 target records show `NULL` for `ENCOUNTER_NUM`.
  - **Forensic Diagnosis:** The ingestion pipeline performs a `LEFT JOIN` between Bronze decimal identifiers and Gold string representations without trimming whitespace or standardizing data types. Records with leading zeros or trailing spaces failed the join condition and were silently dropped by downstream inner filters.
  - **Downstream Business Impact:** High. 34,189 patient encounters are unlinked from billing and medical history marts.

- **[WARNING] Semantic Drift on Flag Padding**
  - **Component / Column:** `ACTIVE_FLAG`
  - **Observed Behavior:** Source uses fixed-width `CHAR(1)`; target uses dynamic `STRING`.
  - **Forensic Diagnosis:** Queries relying on string equality (`ACTIVE_FLAG = 'Y '`) or downstream drivers expecting fixed byte allocations risk false negative evaluations if spaces are inconsistently trimmed during parquet serialization.

### 4. Remediation Action Plan & Code

#### Quarantine Isolation Script (PySpark)
Run in a Microsoft Fabric Notebook to isolate the 34,189 dropped records into an audit quarantine table:

```python
from pyspark.sql import functions as F

df_source = spark.read.table("bronze.patient_mapping_raw")
df_target = spark.read.table("gold.patient_mapping")

df_missing = df_source.filter(
    ~F.trim(F.col("ENCOUNTER_NUM").cast("string")).isin(
        df_target.filter(F.col("ENCOUNTER_NUM").isNotNull())
                 .select(F.trim(F.col("ENCOUNTER_NUM")))
                 .rdd.flatMap(lambda x: x).collect()
    )
)

df_missing.write.format("delta").mode("overwrite").saveAsTable("audit.quarantine_dropped_encounters")

#### Production Join Fix
Update the transformation pipeline to cast join keys to explicit types and normalize whitespace:

```python
df_clean = df_bronze.withColumn(
    "CLEAN_ENCOUNTER_NUM", 
    F.coalesce(F.trim(F.col("ENCOUNTER_NUM").cast("string")), F.lit("UNKNOWN"))
)

df_reconciled = df_clean.join(
    df_lookup,
    F.trim(df_clean["CLEAN_ENCOUNTER_NUM"]) == F.trim(df_lookup["ENCOUNTER_NUM"].cast("string")),
    how="left"
)