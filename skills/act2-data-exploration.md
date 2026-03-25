# Skill: Explore CDC Data and Propose Modeling Plan

## Purpose

You are helping a user explore raw CDC data that just landed in Snowflake via an Openflow SQL Server connector. Your job is to **profile the data, map relationships, check quality, and propose a concrete modeling plan** — then STOP and wait for the user to approve before building anything.

This is Act 2 of a live presentation. Act 1 deployed the Openflow connector. Act 3 will build dbt models based on the plan you propose here. You are the bridge between raw ingestion and transformation.

## Environment

| Parameter | Value |
|-----------|-------|
| Source Database (Snowflake) | `CDC_TARGET` |
| Source Schema | `CARDOPS` |
| Role | `SYSADMIN` |
| Warehouse | `CDC_WH` |
| dbt Target (for plan only) | `CDC_TARGET.ANALYTICS` |

### Identifiers

All schema and table names are uppercase — no quoting required:

```sql
SELECT * FROM CDC_TARGET.CARDOPS.ACCOUNTS LIMIT 10;
SELECT * FROM CDC_TARGET.CARDOPS.TRANSACTIONS LIMIT 10;
```

## Source Tables

These 5 tables were replicated from a SQL Server `CardOpsDB` database via Openflow CDC:

| Table | Description | CDC Pattern |
|-------|-------------|-------------|
| `ACCOUNTS` | Customer card accounts — balances, status, risk tier | Slow INSERTs + frequent UPDATEs (balance/status) |
| `MERCHANTS` | Merchant reference data — name, category, location | Slow INSERTs + rare UPDATEs |
| `TRANSACTIONS` | Card authorization events — the high-volume firehose | High-volume INSERTs only |
| `DISPUTES` | Dispute cases with lifecycle progression | Medium INSERTs + frequent UPDATEs |
| `FRAUDALERTS` | Fraud detection events from scoring engine | Medium INSERTs only |

### Journal Tables

Openflow also creates journal tables (named `*_JOURNAL_*`) alongside each base table. These contain the raw CDC change stream. **Acknowledge them but focus your exploration on the base tables.** The journal tables are internal to the Openflow merge process.

---

## Phase 1: Schema Profiling

Run these checks for **each** of the 5 base tables. Present findings in a clean summary.

### 1a. Row Counts

```sql
SELECT 'ACCOUNTS' AS table_name, COUNT(*) AS row_count FROM CDC_TARGET.CARDOPS.ACCOUNTS
UNION ALL
SELECT 'MERCHANTS', COUNT(*) FROM CDC_TARGET.CARDOPS.MERCHANTS
UNION ALL
SELECT 'TRANSACTIONS', COUNT(*) FROM CDC_TARGET.CARDOPS.TRANSACTIONS
UNION ALL
SELECT 'DISPUTES', COUNT(*) FROM CDC_TARGET.CARDOPS.DISPUTES
UNION ALL
SELECT 'FRAUDALERTS', COUNT(*) FROM CDC_TARGET.CARDOPS.FRAUDALERTS
ORDER BY table_name;
```

### 1b. Column Inventory

For each table, run:

```sql
SELECT
    column_name,
    data_type,
    is_nullable,
    character_maximum_length,
    numeric_precision,
    numeric_scale
FROM CDC_TARGET.information_schema.columns
WHERE table_schema = 'CARDOPS'
  AND table_name = '<TABLE_NAME>'
ORDER BY ordinal_position;
```

### 1c. Null Rates and Distinct Counts

For each table, generate and run a query that checks null rate and distinct count for every column:

```sql
SELECT
    '<column>' AS column_name,
    COUNT(*) AS total_rows,
    COUNT("<column>") AS non_null,
    COUNT(*) - COUNT("<column>") AS null_count,
    ROUND(100.0 * (COUNT(*) - COUNT("<column>")) / NULLIF(COUNT(*), 0), 1) AS null_pct,
    COUNT(DISTINCT "<column>") AS distinct_count
FROM CDC_TARGET.CARDOPS.<TABLE_NAME>;
```

Build this dynamically for all columns in a single query using UNION ALL.

### Present Findings

After profiling, summarize each table:
- Row count
- Column count
- Key columns and their types
- Notable null patterns (any columns with >5% nulls)
- Cardinality observations (high/low distinct counts relative to row count)

---

## Phase 2: Relationship Mapping

The CDC tables have no foreign key constraints in Snowflake — you need to **discover** the join paths yourself from the column metadata and actual data.

### 2a. Discover Potential Join Keys

Look at the column names and types across all 5 tables from Phase 1. Identify columns that appear in multiple tables (shared naming patterns like `*ID` columns, matching data types). These are your candidate join keys.

### 2b. Validate Candidate Joins

For each candidate relationship you identify:
1. Confirm the column values actually overlap between the two tables (join and check match rates)
2. Determine the cardinality — is it one-to-one, one-to-many, or many-to-many?
3. Check for orphaned records (rows in the child table with no matching parent) — some orphans are normal in CDC due to timing gaps

### 2c. Quality Checks

While exploring the data, also check for:
- **Duplicate primary keys** — CDC can sometimes produce dupes during merge windows
- **Data freshness** — How recent is the latest record in each table? Flag anything that looks stale if the source system is actively generating data
- **Unexpected values** — Spot-check categorical/enum columns for values that don't look right
- **Null patterns** — Any columns that are unexpectedly null or unexpectedly never null

Report any quality issues alongside your relationship findings.

### 2d. Draw the Relationship Map

Present the discovered relationships as a visual diagram showing:
- Which tables connect to which
- The join keys used
- The cardinality of each relationship
- Any orphan counts that indicate CDC timing gaps
- Any quality concerns discovered

---

## Phase 3: Data Distribution Analysis

Based on what you learned in Phases 1 and 2, dig deeper into the **most interesting patterns** in the data. You decide what to analyze — use your profiling results and relationship map to guide which distributions, breakdowns, and aggregations will be most insightful.

Focus on:
- How key categorical columns are distributed (status fields, type fields, category fields)
- Numeric ranges and averages for important measures
- Volume patterns across related tables (e.g., how event tables relate to their reference tables)
- Any skew, outliers, or surprising patterns worth calling out

Present your findings as concise tables or summaries. Highlight what's interesting or what would influence how you'd model this data downstream.

---

## Phase 4: Synthesis and Modeling Proposal

### STOP HERE AND PRESENT TO THE USER

After completing Phases 1-3, synthesize your findings and propose a **concrete modeling plan**. Present this as a structured proposal and **wait for user approval before proceeding**.

### Synthesis Format

```
## Data Exploration Summary

### What We Found
- [Row counts, table sizes]
- [Relationship integrity status]
- [Key distribution insights]
- [Quality assessment]

### Key Observations
- [2-3 bullet points about what's interesting or notable in the data]
- [Any data patterns that suggest specific modeling approaches]
- [CDC-specific observations: are updates flowing? is the firehose table growing?]
```

### Modeling Proposal Format

Propose concrete models with this level of detail for each:

```
## Proposed Modeling Plan

Target schema: CDC_TARGET.ANALYTICS

### Staging Layer (1:1 clean copies)

| Model Name | Source Table | Key Transformations |
|-----------|-------------|---------------------|
| stg_accounts | CARDOPS.ACCOUNTS | [rename columns, cast types, add surrogate key] |
| stg_merchants | CARDOPS.MERCHANTS | [...] |
| stg_transactions | CARDOPS.TRANSACTIONS | [...] |
| stg_disputes | CARDOPS.DISPUTES | [...] |
| stg_fraud_alerts | CARDOPS.FRAUDALERTS | [...] |

### Mart Layer (business-facing models)

| Model Name | Type | Grain | Key Columns | Joins |
|-----------|------|-------|-------------|-------|
| fct_transactions | Fact | One row per transaction | [list key columns] | Joins to dim_accounts, dim_merchants |
| fct_disputes | Fact | One row per dispute | [...] | Joins to dim_accounts, fct_transactions |
| fct_fraud_alerts | Fact | One row per alert | [...] | Joins to fct_transactions, dim_accounts |
| dim_accounts | Dimension | One row per account (SCD Type 1) | [...] | - |
| dim_merchants | Dimension | One row per merchant | [...] | - |

### Downstream Use Cases These Models Enable:
- [List 3-5 business questions these models will answer]
- [Reference the Act 6 payoff questions if applicable]
```

### Critical: Wait for Approval

After presenting the modeling plan, say:

> "This is the modeling plan I'd propose based on what the data shows. Want me to proceed with building this in dbt, or would you like to adjust anything first?"

**DO NOT proceed to write any dbt code, SQL files, or YAML until the user explicitly approves.**

---

## Phase 5: dbt Build (After Approval)

Once the user approves the modeling plan, build the dbt project. Before writing any models, create the following macro to prevent dbt from generating `ANALYTICS_ANALYTICS` as the schema name:

### Schema Fix (Required)

Create `macros/generate_schema_name.sql` in the dbt project:

```sql
{% macro generate_schema_name(custom_schema_name, node) -%}
    {%- if custom_schema_name is none -%}
        {{ default_schema }}
    {%- else -%}
        {{ custom_schema_name | trim }}
    {%- endif -%}
{%- endmacro %}
```

> **Why:** dbt's default behavior concatenates `target.schema` + model `schema` config. If both are `ANALYTICS`, you get `ANALYTICS_ANALYTICS`. This macro uses the custom schema name directly.

Create this macro **before** running any `dbt build` or `dbt run` commands.

---

## Presentation Tips

- This is a live demo. Keep your SQL output summaries **concise and visual** — use tables, not walls of numbers.
- When presenting findings, narrate what's interesting: "The transaction firehose is generating ~X rows per minute" or "Disputes show a healthy lifecycle funnel with most moving from OPENED to RESOLVED."
- The audience insight is: CoCo doesn't just describe data — it **reasons about how to model it**.
- The transition to Act 3 should feel natural: "Now that we have a plan, let's build it."

---

## Troubleshooting

| Symptom | Cause | Fix |
|---------|-------|-----|
| `Schema 'CDC_TARGET.CARDOPS' does not exist` | Schema not created yet | Check Openflow flow status — wait for initial snapshot to start |
| Tables not found | Initial snapshot not complete | Check Openflow flow status — wait for snapshot to finish |
| Zero rows in tables | CDC flow not started or errored | Check Openflow bulletins for errors |
| Very stale timestamps | Simulator not running | Start the CardOpsDB simulator to generate fresh CDC events |
| Journal tables appear in profiling | Normal Openflow behavior | Ignore `*_JOURNAL_*` tables — focus on base tables only |
