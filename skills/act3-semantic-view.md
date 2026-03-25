# Skill: Build Semantic View with dbt_semantic_view Package

## Purpose

You just finished Act 2 — you explored the CDC data, built a star schema with dbt, and analyzed the resulting mart models. You know the tables, relationships, distributions, and business questions this data can answer.

Now you're going to take everything you learned and encode it as a **Snowflake Semantic View** using the `dbt_semantic_view` package. This creates a first-class Snowflake object that defines relationships, facts, dimensions, and metrics — the business logic layer that a Cortex Agent will use to answer natural language questions.

**Your Act 2 analysis IS the design input.** Use the tables you built, the joins you validated, the distributions you profiled, and the business insights you surfaced to drive every decision in this semantic view.

**You MUST use the `dbt_semantic_view` package to create the semantic view as a dbt model.** Do NOT write raw SQL `CREATE SEMANTIC VIEW` statements. The semantic view is created by adding a new dbt model file with `materialized='semantic_view'` and running `dbt build`.

## Environment

| Parameter | Value |
|-----------|-------|
| Role | `SYSADMIN` |
| Warehouse | `CDC_WH` |
| Database | `CDC_TARGET` |
| Target Schema | `ANALYTICS` |
| dbt Package | `Snowflake-Labs/dbt_semantic_view` v1.0.3 |
| dbt Project | The same project you built in Act 2 |

---

## Step 1: Add the dbt_semantic_view Package

Add to your project's `packages.yml` (create it if it doesn't exist):

```yaml
packages:
  - package: Snowflake-Labs/dbt_semantic_view
    version: "1.0.3"
```

Then install the package by running `dbt deps`.

---

## Step 2: Fix Schema Generation (Prevent ANALYTICS_ANALYTICS)

> **CRITICAL:** dbt's default `generate_schema_name` macro concatenates `target.schema` + model `schema` config. If both are `ANALYTICS`, you get `ANALYTICS_ANALYTICS`.

If `macros/generate_schema_name.sql` does NOT already exist in the project, create it:

```sql
{% macro generate_schema_name(custom_schema_name, node) -%}
    {%- if custom_schema_name is none -%}
        {{ default_schema }}
    {%- else -%}
        {{ custom_schema_name | trim }}
    {%- endif -%}
{%- endmacro %}
```

---

## Step 3: Create the Semantic View Model

Create a new `.sql` file in your models directory (e.g., `models/marts/cardops_semantic_view.sql`).

The model uses a special materialization provided by the package:

```sql
{{ config(materialized='semantic_view') }}
```

The rest of the file is **not SQL** — it's a declarative definition using these clauses:

### TABLES — Register your mart models

Alias each model you want in the semantic view using `{{ ref() }}`:

```sql
TABLES(
    t1 AS {{ ref('my_fact_table') }},
    t2 AS {{ ref('my_dimension_table') }}
)
```

**Design guidance:** Include all fact and dimension tables from your star schema. Use short aliases — you'll reference them throughout.

### RELATIONSHIPS — Define join paths

Tell the semantic view how tables connect:

```sql
RELATIONSHIPS(
    t1(foreign_key_col) REFERENCES t2(primary_key_col)
)
```

**Design guidance:** These are the FK relationships you discovered and validated in Act 2. Every fact-to-dimension join you tested should be a RELATIONSHIP here.

### FACTS — Numeric measures

Declare which columns are measurable values. The syntax is `<alias>.<LOGICAL_NAME> AS <COLUMN_NAME>` where the right side of `AS` is the **actual column name** from the underlying table:

```sql
FACTS(
    t1.TXN_AMOUNT AS AMOUNT,
    t1.RISK_SCORE AS ALERT_SCORE
)
```

> **IMPORTANT:** Do NOT use `AS value`. The word `value` is not a valid column reference and will cause an `invalid identifier 'VALUE'` compilation error. The right side of `AS` must be the actual column name from the underlying table.

**Design guidance:** These are the numeric columns from your fact tables — amounts, scores, counts, flags (0/1 indicators). If you aggregated it in your Act 2 analysis, it's probably a fact.

### DIMENSIONS — Attributes for filtering and grouping

Declare categorical and temporal columns. Same syntax as FACTS — `<alias>.<LOGICAL_NAME> AS <COLUMN_NAME>` where the right side is the **actual column name**:

```sql
DIMENSIONS(
    t1.TXN_STATUS AS AUTH_DECISION,
    t1.TXN_CHANNEL AS CHANNEL,
    t2.MERCHANT_CATEGORY AS CATEGORY_NAME,
    t1.TXN_DATE AS AUTH_DATETIME
)
```

> **IMPORTANT:** Do NOT use `AS value`. Use the actual column name from the underlying table.

**Design guidance:** These are the columns you used for GROUP BY, WHERE filters, and breakdowns in your Act 2 analysis. Status fields, category fields, risk tiers, regions, timestamps — all dimensions.

### METRICS — Pre-defined aggregations

Define named calculations the agent can use directly:

```sql
METRICS(
    t1.total_amount AS SUM(t1.amount),
    t1.avg_amount AS AVG(t1.amount),
    t1.record_count AS COUNT(t1.id),
    t1.high_score_count AS COUNT_IF(t1.score > 80)
)
```

**Design guidance:** Think about the business questions from your Act 2 analysis. Each insight you surfaced — "approval rate by channel," "disputes per risk tier," "top merchants by volume" — implies one or more metrics. Define them here so the Cortex Agent can answer those questions directly.

### COMMENT — Describe the semantic view

```sql
COMMENT='Description of what this semantic view covers and its purpose.'
```

### Full Example (Generic)

```sql
{{ config(materialized='semantic_view') }}

TABLES(
    t1 AS {{ ref('base_table') }},
    t2 AS {{ source('my_source', 'other_table') }}
)
DIMENSIONS(
    t1.ROW_COUNT AS COUNT,
    t2.TRADE_VOLUME AS VOLUME
)
METRICS(
    t1.total_rows AS SUM(t1.count),
    t2.max_volume AS MAX(t2.volume)
)
COMMENT='example semantic view'
```

> **Note:** Not all clauses are required. At minimum you need `TABLES()` and at least one of `FACTS()`, `DIMENSIONS()`, or `METRICS()`. But for a Cortex Agent to be useful, you want all five clauses populated.

---

## Step 4: Design Your Semantic View from Act 2 Analysis

Use this checklist to translate your Act 2 work into semantic view clauses:

1. **TABLES** — List every mart model (fact + dimension) you built. These are your `{{ ref() }}` targets.
2. **RELATIONSHIPS** — Map every validated FK join. Fact → Dimension, Fact → Fact (if applicable).
3. **FACTS** — Every numeric column you aggregated in your analysis (amounts, scores, boolean flags).
4. **DIMENSIONS** — Every column you grouped by or filtered on (statuses, categories, channels, risk tiers, regions, timestamps).
5. **METRICS** — For each business insight or question from your analysis, define the aggregation:
   - "Transaction volume by channel" → `SUM(amount)`, `COUNT(id)` with channel as dimension
   - "Approval rate" → needs a `COUNT_IF` or conditional metric
   - "Average score by rule" → `AVG(score)` with rule as dimension
   - "Disputes per account" → `COUNT(dispute_id)` with account dimension
6. **COMMENT** — One sentence describing the domain and purpose.

---

## Step 5: Build and Validate

Run `dbt build` selecting your semantic view model.

Then validate in Snowflake:

```sql
SHOW SEMANTIC VIEWS IN SCHEMA CDC_TARGET.ANALYTICS;

DESCRIBE SEMANTIC VIEW CDC_TARGET.ANALYTICS.<YOUR_SEMANTIC_VIEW_NAME>;
```

Quick query test:

```sql
SELECT *
FROM SEMANTIC_VIEW(CDC_TARGET.ANALYTICS.<YOUR_SEMANTIC_VIEW_NAME>
    METRICS <alias>.<metric_name>
    DIMENSIONS <alias>.<dimension_name>
);
```

---

## Troubleshooting

| Symptom | Likely Cause | Fix |
|---------|-------------|-----|
| `dbt deps` fails to find package | Wrong package name or version | `Snowflake-Labs/dbt_semantic_view` version `"1.0.3"` |
| `Materialization 'semantic_view' not found` | Package not installed | Run `dbt deps` |
| Model materializes in `ANALYTICS_ANALYTICS` | Missing `generate_schema_name` macro | Create `macros/generate_schema_name.sql` per Step 2 |
| `Object does not exist` in TABLES clause | Mart models not built or wrong schema | Run `dbt build` for mart models first |
| Column name mismatch | Case sensitivity | Snowflake stores unquoted identifiers as uppercase — use `AMOUNT` not `amount` |
| `invalid identifier 'VALUE'` | Wrong FACTS/DIMENSIONS syntax | The right side of `AS` must be the actual column name, not the word `value`. Use `t1.TXN_AMOUNT AS AMOUNT`, not `t1.AMOUNT AS value` |
| `RELATIONSHIPS` syntax error | Wrong format | `alias(COLUMN) REFERENCES other_alias(COLUMN)` — no `ON` keyword |
| `SHOW SEMANTIC VIEWS` returns nothing | Wrong schema context | Use `SHOW SEMANTIC VIEWS IN SCHEMA CDC_TARGET.ANALYTICS` |
| Semantic view query returns no data | Empty underlying tables | Check mart model row counts |
