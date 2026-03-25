# Build Faster with Cortex Code

**Virtual Hands-On Lab**

---

## Overview

A single AI assistant guides you from raw source data to conversational business answers — no documentation hunting, no boilerplate, no context switching. Cortex Code meets you where you work: in the CLI for infrastructure, in Snowsight for everything else.

What used to take days of stitching together pipelines, writing transformation logic, and building interfaces collapses into one continuous, AI-assisted workflow.

---

## Prerequisites

- A Snowflake account with access to **Cortex Code** (CLI and/or Snowsight)
- A warehouse (e.g., `DEVELOPMENT_WH`) and the `SYSADMIN` role
- *(Optional for Act 1)* A **SQL Server 2022** instance with Change Tracking enabled — see [`configs.md`](configs.md) for required parameters

> **Don't have a SQL Server instance?** No problem. Run the setup notebook (`notebook/00_hol_setup.ipynb`) to bootstrap the dataset directly, then start at Act 2.

---

## What You'll Build

| Act | Interface | What You Do |
|-----|-----------|-------------|
| **Act 1** | CLI | Deploy an Openflow CDC connector to ingest data from SQL Server into Snowflake |
| **Act 2** | Snowsight | Explore the data, profile relationships, and build a full dbt project (staging + marts) |
| **Act 3** | Snowsight | Define a Semantic View — facts, dimensions, metrics, and relationships the business understands |
| **Act 4** | Snowsight | Create a Cortex Agent wired to the semantic view |
| **Act 5** | Snowflake Intelligence | Chat with your data in plain English — the full circle |

---

## Act 1: Ingestion — Openflow SQL Server Connector

**Interface:** CLI

Deploy an Openflow SQL Server connector using Cortex Code CLI. Raw data from a source system begins flowing into Snowflake.

- Open the terminal and have a conversational interaction with Cortex Code
- Describe what you need — Cortex Code scaffolds the connector configuration
- Deploy the Openflow runtime and connector in one flow
- Data from SQL Server begins landing in Snowflake automatically

> **Skill guide:** [`skills/act1-openflow-sqlserver.md`](skills/act1-openflow-sqlserver.md)

---

## Act 2: Explore, Model & Transform

**Interface:** Snowsight

Switch to Snowsight. Use Cortex Code to explore the ingested data, understand its structure, co-create a modeling plan, and build a full dbt project — all in one continuous conversation.

- Data has landed from the Openflow pipeline (or from the setup notebook)
- Ask Cortex Code to examine tables, profile the data, and surface relationships
- Cortex Code proposes a modeling plan based on what it finds
- Review and confirm the plan before anything gets built — you stay in the loop
- Cortex Code generates the full dbt project: staging models, mart models, YAML configs, and ref() chains
- Run `dbt build` to materialize the star schema in `CDC_TARGET.ANALYTICS`

> **Skill guide:** [`skills/act2-data-exploration.md`](skills/act2-data-exploration.md)

---

## Act 3: Semantic View — Business Logic

**Interface:** Snowsight

Define a Semantic View on top of the dbt-materialized tables using the `dbt_semantic_view` package. This is where raw columns become named facts, dimensions, metrics, and relationships that the business understands.

- Install the `dbt_semantic_view` package and create a new dbt model
- Define tables, relationships, facts, dimensions, and metrics declaratively
- Run `dbt build` to materialize the semantic view as a first-class Snowflake object
- Business logic is now version-controlled, testable, and CI/CD-ready

> **Skill guide:** [`skills/act3-semantic-view.md`](skills/act3-semantic-view.md)

---

## Act 4: Cortex Agent — Wire the Interface

**Interface:** Snowsight

Create a Cortex Agent that uses the semantic view to answer natural language questions. This is where all the upstream work — ingestion, modeling, semantic definition — converges into a conversational interface.

- Define and deploy the Cortex Agent using `CREATE AGENT` with a YAML specification
- Point the agent at the semantic view built in Act 3
- Configure sample questions so users see suggested prompts immediately
- The agent is deployed to Snowflake Intelligence and live-queryable

> **Skill guide:** [`skills/act4-cortex-agent.md`](skills/act4-cortex-agent.md)

---

## Act 5: The Payoff — Chat with Your Data

**Interface:** Snowflake Intelligence

Open Snowflake Intelligence and chat with the data. The full circle is complete.

- Ask business questions in natural language
- The agent returns answers grounded in modeled, semantically-defined data
- Anyone in the org — not just engineers — can get answers now

**SQL Server → Openflow → dbt → Semantic View → Cortex Agent → Answers**

---

## Repository Structure

```
.
├── README.md                  ← You are here
├── configs.md                 ← Connector configuration reference
├── docs/
│   ├── data-model.md          ← Source database schema documentation
│   └── demo-roadmap.html      ← Visual demo overview (open in browser)
├── notebook/
│   └── 00_hol_setup.ipynb     ← Dataset bootstrap (skip Act 1)
└── skills/
    ├── act1-openflow-sqlserver.md
    ├── act2-data-exploration.md
    ├── act3-semantic-view.md
    └── act4-cortex-agent.md
```
