# Openflow Connector Configuration

This file describes the parameters needed to deploy an Openflow SQL Server CDC connector for the **Build Faster with Cortex Code** HOL.

## Parameters

| Parameter | Description |
|-----------|-------------|
| **SQLServer Connection URL** | JDBC connection string for your SQL Server instance. Includes host, port, database name, and encryption settings. |
| **SQLServer Username** | SQL Server login username with read access and Change Tracking permissions on the source tables. |
| **SQLServer Password** | Password for the SQL Server login. If it contains special characters (`\`, `!`, etc.), use the Python method in the Act 1 skill guide to avoid shell escaping issues. |
| **Destination Database** | The Snowflake database where replicated tables will land. Created before running the connector. |
| **Snowflake Warehouse** | Warehouse used by the Openflow connector for writing data into Snowflake. |
| **Snowflake Role** | Role with the necessary grants on the destination database, warehouse, and Openflow runtime. |
| **Snowflake Authentication Strategy** | How the SPCS runtime authenticates to Snowflake. Use `SNOWFLAKE_MANAGED` for SPCS deployments. |
| **Included Table Names** | Comma-separated list of fully qualified table names (schema + table) to replicate. Each entry uses `"SCHEMA"."TABLE"` format. |
| **Ingestion Type** | The replication strategy. Use `cdc` for Change Tracking-based incremental replication. |

## Example Configuration

Replace the placeholder values with your own environment details:

```json
{
  "SQLServer Connection URL": "jdbc:sqlserver://<SQLSERVER_HOST>:<SQLSERVER_PORT>;databaseName=<SOURCE_DATABASE>;encrypt=false",
  "SQLServer Username": "<SQLSERVER_USER>",
  "SQLServer Password": "<SQLSERVER_PASSWORD>",
  "Destination Database": "CDC_TARGET",
  "Snowflake Warehouse": "CDC_WH",
  "Snowflake Role": "OPENFLOW_ADMIN",
  "Snowflake Authentication Strategy": "SNOWFLAKE_MANAGED",
  "Included Table Names": "\"CARDOPS\".\"ACCOUNTS\", \"CARDOPS\".\"DISPUTES\", \"CARDOPS\".\"FRAUDALERTS\", \"CARDOPS\".\"MERCHANTS\", \"CARDOPS\".\"TRANSACTIONS\"",
  "Ingestion Type": "cdc"
}
```
