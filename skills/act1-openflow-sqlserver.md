# Skill: Deploy SQL Server CDC Connector in Openflow

> **Note for HOL attendees:** The SQL Server instance used during the live demo is not publicly accessible. If you have your own **SQL Server 2022** environment with Change Tracking enabled, you can follow this guide to deploy an Openflow CDC connector against it. Fill in the [Configuration](#configuration) table below with your own connection details and you're good to go. If you don't have a SQL Server instance, skip to Act 2 — the setup notebook (`00_hol_setup.ipynb`) pre-loads the same dataset so you can continue the lab without ingestion.

## Purpose

Step-by-step guide for deploying an Openflow SQL Server CDC connector that replicates `CardOpsDB` (5 tables) into Snowflake. Built for the **Build Faster with Cortex Code** HOL — Act 1: Ingestion.

## Configuration

Before starting, fill in your environment-specific values. These placeholders are used throughout this guide.

| Variable | Description | Example |
|----------|-------------|---------|
| `<RUNTIME_NAME>` | Your Openflow runtime name | `MY_CDC_RUNTIME` |
| `<RUNTIME_KEY>` | Lowercase, no-underscore runtime key (used in URLs) | `mycdcruntime` |
| `<RUNTIME_INTEGRATION>` | Runtime integration identifier | `OPENFLOW_RUNTIME_XXXXXXXX_...` |
| `<NIPYAPI_PROFILE>` | Local nipyapi profile name you'll create | `my_profile` |
| `<SNOWFLAKE_ACCOUNT>` | Your Snowflake account locator | `MYORG-MYACCOUNT` |
| `<SQLSERVER_HOST>` | SQL Server IP or hostname | `10.0.0.5` |
| `<SQLSERVER_PORT>` | SQL Server port | `1433` |
| `<SQLSERVER_USER>` | SQL Server login username | `sa` |
| `<SQLSERVER_PASSWORD>` | SQL Server login password | *(your password)* |
| `<SOURCE_DATABASE>` | Source database name on SQL Server | `CardOpsDB` |
| `<SOURCE_SCHEMA>` | Source schema name | `CARDOPS` |
| `<DEST_DATABASE>` | Snowflake destination database | `CDC_TARGET` |
| `<DEST_WAREHOUSE>` | Snowflake warehouse for ingestion | `CDC_WH` |
| `<OPENFLOW_ROLE>` | Snowflake role with Openflow grants | `OPENFLOW_ADMIN` |

## Environment

| Parameter | Value |
|-----------|-------|
| Openflow Runtime | `<RUNTIME_NAME>` |
| Runtime Integration | `<RUNTIME_INTEGRATION>` |
| Runtime Key (URL segment) | `<RUNTIME_KEY>` |
| nipyapi Profile | `<NIPYAPI_PROFILE>` |
| Deployment Type | SPCS |
| Snowflake Account | `<SNOWFLAKE_ACCOUNT>` |
| Snowflake Role | `<OPENFLOW_ROLE>` |
| Source Host | `<SQLSERVER_HOST>` |
| Source Port | `<SQLSERVER_PORT>` |
| Source Database | `<SOURCE_DATABASE>` |
| Source Schema | `<SOURCE_SCHEMA>` |
| Source User | `<SQLSERVER_USER>` |
| Source Password | *(set via CLI — see Step 3)* |
| Destination Database | `<DEST_DATABASE>` |
| Destination Warehouse | `<DEST_WAREHOUSE>` |
| Auth Strategy | `SNOWFLAKE_MANAGED` |
| Connector Flow | `sqlserver` |
| Registry | `ConnectorFlowRegistryClient` |
| JDBC Driver | `mssql-jdbc-12.10.0.jre11.jar` |

### Tables Being Replicated

| Table | CDC Pattern |
|-------|------------|
| `ACCOUNTS` | Slow INSERTs + UPDATEs (balance/status) |
| `MERCHANTS` | Slow INSERTs + rare UPDATEs |
| `TRANSACTIONS` | High-volume INSERTs (firehose) |
| `DISPUTES` | Medium INSERTs + frequent UPDATEs (lifecycle) |
| `FRAUDALERTS` | Medium INSERTs (event stream) |

---

## Pre-Flight Checklist

Before starting, confirm:

- [ ] nipyapi is installed and ≥ 1.2.0
- [ ] SQL Server at `<SQLSERVER_HOST>:<SQLSERVER_PORT>` is reachable
- [ ] Snowflake database `<DEST_DATABASE>` and warehouse `<DEST_WAREHOUSE>` exist
- [ ] Role `<OPENFLOW_ROLE>` has necessary grants

### Quick Validation Commands

```bash
nipyapi --version
# Expected: 1.5.0+
```

---

## Step 0: Create nipyapi Profile for <RUNTIME_NAME>

> The nipyapi profile maps a local name to a runtime's NiFi API URL + bearer token. This must exist before any `nipyapi` commands will work.

### 0a. Get a Bearer Token

The bearer token authenticates nipyapi to the SPCS runtime. Reuse an existing token from another profile on the same account, or generate one:

```bash
# Option A: Reuse token from another profile on the same account
grep "nifi_bearer_token" ~/.nipyapi/profiles.yml | head -1

# Option B: Check for PAT in environment
echo $SNOWFLAKE_PAT

# Option C: Get from Snowsight → Profile → Personal Access Tokens
```

### 0b. Create the Profile

```bash
cat >> ~/.nipyapi/profiles.yml << 'EOF'
<NIPYAPI_PROFILE>:
  nifi_url: "https://of--<SNOWFLAKE_ACCOUNT>.snowflakecomputing.app/<RUNTIME_KEY>/nifi-api"
  nifi_bearer_token: "<PASTE_TOKEN_HERE>"
EOF
```

> Replace `<SNOWFLAKE_ACCOUNT>` with your account locator (lowercase, hyphen-separated). Example: `sfsenorthamerica-my_account` becomes `sfsenorthamerica-my_account`.

### 0c. Verify Connectivity

```bash
nipyapi --profile <NIPYAPI_PROFILE> system get_nifi_version_info
# Expected: Returns NiFi version (e.g., 2026.3.19.18)

nipyapi --profile <NIPYAPI_PROFILE> ci list_flows
# Expected: empty process_groups array (clean runtime)
```

### Checkpoint

- [ ] Profile exists in `~/.nipyapi/profiles.yml`
- [ ] `get_nifi_version_info` returns a version
- [ ] `list_flows` returns empty (no existing deployments)

---

## Step 1: Network Access (EAI)

> SPCS runtimes cannot reach external hosts without an External Access Integration.

### 1a. Create Network Rule

> Network rules require a database.schema context. We use `<DEST_DATABASE>.OPENFLOW_CONFIG`.

```sql
USE ROLE <OPENFLOW_ROLE>;
USE DATABASE <DEST_DATABASE>;
CREATE SCHEMA IF NOT EXISTS <DEST_DATABASE>.OPENFLOW_CONFIG;

CREATE OR REPLACE NETWORK RULE <DEST_DATABASE>.OPENFLOW_CONFIG.SQLSERVER_CDC_NETWORK_RULE
  TYPE = HOST_PORT
  MODE = EGRESS
  VALUE_LIST = ('<SQLSERVER_HOST>:<SQLSERVER_PORT>');
```

### 1b. Create EAI

> **Requires ACCOUNTADMIN.** EAI creation is an account-level operation — `<OPENFLOW_ROLE>` and `SECURITYADMIN` do not have sufficient privileges.

```sql
USE ROLE ACCOUNTADMIN;

CREATE OR REPLACE EXTERNAL ACCESS INTEGRATION SQLSERVER_CDC_EAI
  ALLOWED_NETWORK_RULES = (<DEST_DATABASE>.OPENFLOW_CONFIG.SQLSERVER_CDC_NETWORK_RULE)
  ENABLED = TRUE
  COMMENT = 'EAI for Openflow SQL Server CDC — <SOURCE_DATABASE> replication';

GRANT USAGE ON INTEGRATION SQLSERVER_CDC_EAI TO ROLE <OPENFLOW_ROLE>;
```

### 1c. Attach EAI to Runtime

> **Manual step** — done in the Openflow Control Plane UI:
> 1. Snowsight → Data → Openflow
> 2. Click on `<RUNTIME_NAME>` runtime
> 3. Edit → Add `SQLSERVER_CDC_EAI` to the External Access Integrations list
> 4. Save — runtime will restart

### Checkpoint

Wait for runtime to return to `RUNNING` state before proceeding.

---

## Step 2: Deploy the Connector Flow

```bash
nipyapi --profile <NIPYAPI_PROFILE> ci deploy_flow \
  --registry_client ConnectorFlowRegistryClient \
  --bucket connectors \
  --flow sqlserver
```

### Capture Output

The deploy command returns a **process group ID**. Save it — every subsequent command needs it.

```bash
# Example output:
# Process group deployed: <PG_ID>
#
# Export it for convenience:
export PG_ID="<paste-process-group-id-here>"
```

### Checkpoint

```bash
nipyapi --profile <NIPYAPI_PROFILE> ci list_flows
# Should show the sqlserver flow in the list
```

---

## Step 3: Configure Parameters

All parameters are set in a **single command** using `configure_inherited_params`. The command auto-routes each parameter to the correct context (Source, Destination, or Ingestion) via the inheritance hierarchy.

> **IMPORTANT — Actual parameter names:**
> - Source params use `SQLServer` (no space), e.g., `SQLServer Connection URL`
> - Destination database is `Destination Database` (not `Snowflake Database`)
> - Auth strategy is `SNOWFLAKE_MANAGED` (not `SNOWFLAKE_SESSION_TOKEN`)
> - These differ from what the Openflow skill references suggest. Always verify with `export_parameters` if unsure.

### 3a. Set All Parameters (except password)

```bash
nipyapi --profile <NIPYAPI_PROFILE> ci configure_inherited_params \
  --process_group_id "$PG_ID" \
  --parameters '{
    "SQLServer Connection URL": "jdbc:sqlserver://<SQLSERVER_HOST>:<SQLSERVER_PORT>;databaseName=<SOURCE_DATABASE>;encrypt=false",
    "SQLServer Username": "<SQLSERVER_USER>",
    "Destination Database": "<DEST_DATABASE>",
    "Snowflake Warehouse": "<DEST_WAREHOUSE>",
    "Snowflake Role": "<OPENFLOW_ROLE>",
    "Snowflake Authentication Strategy": "SNOWFLAKE_MANAGED",
    "Included Table Names": "\"<SOURCE_SCHEMA>\".\"ACCOUNTS\", \"<SOURCE_SCHEMA>\".\"DISPUTES\", \"<SOURCE_SCHEMA>\".\"FRAUDALERTS\", \"<SOURCE_SCHEMA>\".\"MERCHANTS\", \"<SOURCE_SCHEMA>\".\"TRANSACTIONS\"",
    "Ingestion Type": "cdc"
  }'
```

**Expected output:** `parameters_updated: 8`, `contexts_modified: 3`

### 3b. Set Password (special handling)

> If your password contains special characters (backslashes, `!`, etc.), shell escaping can be unreliable. Use Python to ensure the exact value is sent:

```bash
python3 -c "
import json, subprocess
params = json.dumps({'SQLServer Password': '<SQLSERVER_PASSWORD>'})
result = subprocess.run([
    'nipyapi', '--profile', '<NIPYAPI_PROFILE>',
    'ci', 'configure_inherited_params',
    '--process_group_id', '$PG_ID',
    '--parameters', params
], capture_output=True, text=True)
print(result.stdout)
"
```

**Expected output:** `parameters_updated: 1`, `contexts_modified: 1`, routed to `SQLServer Source Parameters`

### Checkpoint

Verify parameters were set correctly:

```bash
nipyapi --profile <NIPYAPI_PROFILE> ci export_parameters \
  --process_group_id "$PG_ID"
```

Confirm `SQLServer Connection URL`, `SQLServer Username`, `Destination Database`, `Snowflake Warehouse`, `Snowflake Role`, `Included Table Names`, and `Ingestion Type` all have non-null values. Password will show as `null` (masked sensitive value) — this is expected.

---

## Step 4: Upload JDBC Driver

The Microsoft SQL Server JDBC driver is **not bundled** with the connector. It must be uploaded as an asset.

> Use `--process_group_id` (not `--context_id`). The command auto-discovers the correct parameter context. The parameter name is `SQLServer JDBC Driver` (no space between SQL and Server).

```bash
nipyapi --profile <NIPYAPI_PROFILE> ci upload_asset \
  --process_group_id "$PG_ID" \
  --url "https://repo1.maven.org/maven2/com/microsoft/sqlserver/mssql-jdbc/12.10.0.jre11/mssql-jdbc-12.10.0.jre11.jar" \
  --param_name "SQLServer JDBC Driver"
```

### Checkpoint

**Expected output:** `parameter_updated: true`, `parameter_name: SQLServer JDBC Driver`, asset linked to `SQLServer Ingestion Parameters` context.

---

## Step 5: Enable Controllers

```bash
nipyapi --profile <NIPYAPI_PROFILE> canvas schedule_all_controllers "$PG_ID" True
```

### Checkpoint

Check for bulletin errors:

```bash
nipyapi --profile <NIPYAPI_PROFILE> ci get_bulletins --process_group_id "$PG_ID"
```

No errors expected. If you see connection errors, the EAI or credentials need fixing.

---

## Step 6: Start the Flow

```bash
nipyapi --profile <NIPYAPI_PROFILE> ci start_flow \
  --process_group_id "$PG_ID"
```

### Checkpoint

```bash
nipyapi --profile <NIPYAPI_PROFILE> ci get_status \
  --process_group_id "$PG_ID"
```

**Expected:**
- `running_processors > 0`
- `stopped_processors = 0`
- `bulletin_errors = 0`

---

## Step 7: Validate Data Landing in Snowflake

### Initial Schema Check

```sql
USE ROLE <OPENFLOW_ROLE>;
USE WAREHOUSE <DEST_WAREHOUSE>;

SHOW SCHEMAS IN DATABASE <DEST_DATABASE>;
SHOW TABLES IN SCHEMA <DEST_DATABASE>.<SOURCE_SCHEMA>;
```

### Row Count Validation

```sql
SELECT 'ACCOUNTS' AS table_name, COUNT(*) AS row_count FROM <DEST_DATABASE>.<SOURCE_SCHEMA>.ACCOUNTS
UNION ALL
SELECT 'MERCHANTS', COUNT(*) FROM <DEST_DATABASE>.<SOURCE_SCHEMA>.MERCHANTS
UNION ALL
SELECT 'TRANSACTIONS', COUNT(*) FROM <DEST_DATABASE>.<SOURCE_SCHEMA>.TRANSACTIONS
UNION ALL
SELECT 'DISPUTES', COUNT(*) FROM <DEST_DATABASE>.<SOURCE_SCHEMA>.DISPUTES
UNION ALL
SELECT 'FRAUDALERTS', COUNT(*) FROM <DEST_DATABASE>.<SOURCE_SCHEMA>.FRAUDALERTS
ORDER BY table_name;
```

**Expected counts (seed data):**

| Table | Expected Rows |
|-------|--------------|
| ACCOUNTS | 50 |
| MERCHANTS | 30 |
| TRANSACTIONS | 500 |
| DISPUTES | 25 |
| FRAUDALERTS | 40 |

### CDC Validation

After confirming initial load, trigger the simulator to generate new rows, then re-check counts to confirm CDC is flowing.

---

## Teardown (Post-Demo)

To clean up after the presentation:

```bash
# Stop the flow
nipyapi --profile <NIPYAPI_PROFILE> ci stop_flow \
  --process_group_id "$PG_ID"

# Disable controllers
nipyapi --profile <NIPYAPI_PROFILE> canvas schedule_all_controllers "$PG_ID" False

# Delete the flow (optional)
nipyapi --profile <NIPYAPI_PROFILE> ci delete_flow \
  --process_group_id "$PG_ID"
```

```sql
-- Drop replicated data (optional)
DROP SCHEMA IF EXISTS <DEST_DATABASE>.<SOURCE_SCHEMA> CASCADE;
```

---

## Troubleshooting Quick Reference

| Symptom | Likely Cause | Fix |
|---------|-------------|-----|
| Controller verification fails on SQL Server | EAI not attached or expired token | Re-attach EAI, check runtime is RUNNING |
| Controller verification fails on Snowflake | Wrong role/DB/WH or insufficient grants | Verify `<OPENFLOW_ROLE>` has access to `<DEST_DATABASE>` and `<DEST_WAREHOUSE>` |
| JDBC driver upload fails | Wrong param name or context | Use `--process_group_id` (not `--context_id`), param name is `SQLServer JDBC Driver` (no space) |
| Processor verification fails | Table names not found | Verify Change Tracking is enabled on all 5 tables in `<SOURCE_DATABASE>` |
| Flow starts but no data lands | Initial snapshot in progress | Wait 2-3 minutes for initial load to complete |
| Bearer token expired | Token TTL exceeded | Regenerate PAT in Snowsight → Profile → Personal Access Tokens, update `~/.nipyapi/profiles.yml` |
| EAI creation fails with "Insufficient privileges" | Wrong role | EAI requires `ACCOUNTADMIN` — `SECURITYADMIN` and `<OPENFLOW_ROLE>` are not sufficient |
| Network rule creation fails "no current database" | Unqualified name | Network rules need fully qualified names: `DATABASE.SCHEMA.RULE_NAME` |
| nipyapi profile not connecting | Wrong runtime key | Runtime key in URL is lowercase no underscores: `<RUNTIME_KEY>` (not `<RUNTIME_NAME>`) |
| `encrypt=false` rejected | SQL Server TLS config | If needed, change to `encrypt=true;trustServerCertificate=true` |
| `Snowflake Private Key Service` fails verification | Expected on SPCS | Non-blocking. This controller is for key-pair auth (BYOC). Ignore on `SNOWFLAKE_MANAGED` deployments |
| Password with special chars (`\!`) not working | Shell escaping | Use Python subprocess to set passwords containing backslashes or special characters (see Step 3b) |
| Parameter names not found | Wrong naming convention | Actual names use `SQLServer` (no space), `Destination Database` (not `Snowflake Database`). Run `export_parameters` to see real names |
| Incremental CDC targets wrong database | Stale controller services from previous deployment | Delete the connector, drop the destination schema, and redeploy fresh. Old deployments leave controller services with cached JDBC URLs |
| Cannot delete flow — "controller service not disabled" | Trying to delete on wrong runtime | Check the error hostname — if it references a different runtime, you're on the wrong one. Stop flow → disable controllers → delete |

---

## Demo Script Notes

**For the live presentation:**

1. Start with pre-flight check (nipyapi version + connectivity) — shows the tooling
2. Skip EAI if already attached from rehearsal — mention it verbally instead
3. Deploy flow → configure params → upload driver — this is the core "CoCo assists infrastructure" moment
4. Enable controllers → start flow — the Check-Act pattern
5. Switch to Snowflake, run the count query — the "data is here" moment
6. Transition to Act 2 (Exploration in Snowsight)

**Timing estimate:** ~5-7 minutes for Steps 2-7 if everything is pre-staged. Step 1 (EAI) adds 2-3 minutes if runtime restart is needed.
