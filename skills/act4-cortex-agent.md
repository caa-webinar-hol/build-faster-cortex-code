# Skill: Create Cortex Agent for Snowflake Intelligence

## Purpose

You just finished Act 3 — you built a Snowflake Semantic View on top of the dbt mart models from Act 2. The semantic view defines the relationships, facts, dimensions, and metrics that translate raw data into business language.

Now you're going to create a **Cortex Agent** that uses that semantic view to answer natural language questions. The agent will be accessible through **Snowflake Intelligence** — the chat interface where anyone in the org can ask questions about the data.

**You MUST create the agent using the `CREATE AGENT` SQL statement.** The agent points at the semantic view from Act 3 and uses `cortex_analyst_text_to_sql` to convert natural language questions into SQL.

## Environment

| Parameter | Value |
|-----------|-------|
| Role | `SYSADMIN` |
| Warehouse | `CDC_WH` |
| Database (Agent) | `SNOWFLAKE_INTELLIGENCE` |
| Schema (Agent) | `AGENTS` |
| Semantic View | Created in Act 3 — `CDC_TARGET.ANALYTICS.<name from Act 3>` |

> **CRITICAL:** Agents used with Snowflake Intelligence **must** live in the `SNOWFLAKE_INTELLIGENCE.AGENTS` schema. This is a hard requirement — agents created elsewhere will not appear in the Snowflake Intelligence UI.

---

## Step 1: Set Up the Snowflake Intelligence Schema

This only needs to be done once per account. Check if it already exists first:

```sql
SHOW SCHEMAS IN DATABASE SNOWFLAKE_INTELLIGENCE;
```

If the database or schema doesn't exist, create them:

```sql
CREATE DATABASE IF NOT EXISTS SNOWFLAKE_INTELLIGENCE;
GRANT USAGE ON DATABASE SNOWFLAKE_INTELLIGENCE TO ROLE PUBLIC;

CREATE SCHEMA IF NOT EXISTS SNOWFLAKE_INTELLIGENCE.AGENTS;
GRANT USAGE ON SCHEMA SNOWFLAKE_INTELLIGENCE.AGENTS TO ROLE PUBLIC;
```

Then grant the ability to create agents:

```sql
GRANT CREATE AGENT ON SCHEMA SNOWFLAKE_INTELLIGENCE.AGENTS TO ROLE SYSADMIN;
```

---

## Step 2: Create the Agent

The `CREATE AGENT` statement takes a **YAML** specification that defines the agent's model, instructions, sample questions, tools, and tool resources.

### Syntax

```sql
CREATE OR REPLACE AGENT SNOWFLAKE_INTELLIGENCE.AGENTS.<agent_name>
  COMMENT = '<description>'
  PROFILE = '{"display_name": "<display name>"}'
  FROM SPECIFICATION $$
  models:
    orchestration: <model_name>

  instructions:
    orchestration: "<tool selection guidance>"
    response: "<tone and format guidance>"
    sample_questions:
      - question: "<example question a user might ask>"
        answer: "<brief description of how the agent would answer>"
      - question: "<another example question>"
        answer: "<brief description>"

  tools:
    - tool_spec:
        type: cortex_analyst_text_to_sql
        name: <tool_name>
        description: "<what this tool does>"

  tool_resources:
    <tool_name>:
      semantic_view: "<fully_qualified_semantic_view_name>"
  $$;
```

### Key Fields

| Field | Purpose |
|-------|---------|
| `models.orchestration` | The LLM that orchestrates tool selection. Use `claude-4-sonnet` or `llama3.3-70B` |
| `instructions.orchestration` | Guides the agent on when/how to use tools |
| `instructions.response` | Controls tone, format, and style of answers |
| `instructions.sample_questions` | Example questions shown to users in the Snowflake Intelligence UI as suggested prompts. Each entry has a `question` and `answer` field. |
| `tools[].tool_spec.type` | `cortex_analyst_text_to_sql` — converts questions to SQL via the semantic view |
| `tools[].tool_spec.name` | A name you choose — must match the key in `tool_resources` |
| `tools[].tool_spec.description` | Describe what data this tool accesses — helps the LLM decide when to use it |
| `tool_resources.<name>.semantic_view` | Fully qualified path to the semantic view from Act 3 |
| `PROFILE` | JSON with `display_name` — what users see in Snowflake Intelligence |
| `COMMENT` | Description stored as metadata |

### Design Guidance

- **`instructions.orchestration`** — Tell the agent what the data covers and when to use the tool. Reference the domain: card transactions, disputes, fraud alerts, accounts, merchants.
- **`instructions.response`** — Set the tone for a business audience. Concise, data-driven, tables over paragraphs.
- **`instructions.sample_questions`** — Use the business questions from your Act 2 analysis. These become the suggested prompts users see when they open the agent in Snowflake Intelligence. Pick 3-5 questions that showcase the semantic view's capabilities across different tables and metrics. Each question should have a brief answer describing how the agent would respond.
- **`tool_spec.description`** — Be specific about what the semantic view contains. This helps the LLM match user questions to the right tool (matters more when agents have multiple tools).
- **`semantic_view`** — Use the exact fully qualified name of the semantic view you created in Act 3.

---

## Step 3: Grant Access

After creating the agent, grant usage so other roles can interact with it through Snowflake Intelligence:

```sql
GRANT USAGE ON AGENT SNOWFLAKE_INTELLIGENCE.AGENTS.<agent_name> TO ROLE PUBLIC;
```

Also ensure the semantic view is accessible:

```sql
GRANT SELECT ON SEMANTIC VIEW CDC_TARGET.ANALYTICS.<semantic_view_name> TO ROLE PUBLIC;
```

---

## Step 4: Validate

### 4a. Confirm the Agent Exists

```sql
SHOW AGENTS IN SCHEMA SNOWFLAKE_INTELLIGENCE.AGENTS;
```

### 4b. Describe the Agent

```sql
DESC AGENT SNOWFLAKE_INTELLIGENCE.AGENTS.<agent_name>;
```

Confirm the specification shows the correct semantic view reference, model, and sample questions.

### 4c. Test in Snowflake Intelligence

1. Go to Snowsight — **AI & ML — Snowflake Intelligence**
2. Your agent should appear in the list
3. The sample questions you defined should appear as suggested prompts
4. Ask a question that exercises the semantic view — try one of your sample questions first, then freestyle

If the agent returns grounded, data-backed answers — you're done.

---

## Teardown (Post-Demo)

```sql
DROP AGENT IF EXISTS SNOWFLAKE_INTELLIGENCE.AGENTS.<agent_name>;
```

---

## Troubleshooting

| Symptom | Likely Cause | Fix |
|---------|-------------|-----|
| Agent not visible in Snowflake Intelligence UI | Created in wrong schema | Must be in `SNOWFLAKE_INTELLIGENCE.AGENTS` — not any other database/schema |
| `CREATE AGENT` fails with permission error | Missing CREATE AGENT grant | `GRANT CREATE AGENT ON SCHEMA SNOWFLAKE_INTELLIGENCE.AGENTS TO ROLE SYSADMIN` |
| Agent can't access semantic view | Missing SELECT grant | `GRANT SELECT ON SEMANTIC VIEW ... TO ROLE PUBLIC` |
| Agent returns "I don't have access to that data" | Warehouse not specified or stopped | Check the warehouse is running. User's default warehouse is used for execution. |
| Agent gives poor or irrelevant answers | Weak instructions or description | Improve `instructions.orchestration` and `tool_spec.description` with domain-specific context |
| `tool_resources` key doesn't match `tool_spec.name` | Name mismatch | The key in `tool_resources` must exactly match the `name` in the corresponding `tool_spec` |
| YAML parse error in specification | Invalid YAML in `FROM SPECIFICATION` | Check indentation (YAML is whitespace-sensitive), ensure strings with special chars are quoted |
| Agent works but answers are too verbose | Response instructions too open | Tighten `instructions.response` — e.g., "Be concise. Use tables. No preamble." |
| Sample questions not showing in UI | Wrong YAML structure | `sample_questions` must be under `instructions`, each entry needs both `question` and `answer` fields |
