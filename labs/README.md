# DE-225: Real-Time CDC with OpenFlow — Lab Overview

## What This Lab Builds

A real-time Change Data Capture (CDC) pipeline that replicates data from a Snowflake Postgres instance into Snowflake tables, managed entirely through a GitOps CI/CD workflow.

```
┌─────────────────┐       ┌──────────────────────┐       ┌─────────────────┐
│  Snowflake      │  CDC  │  OpenFlow Runtime     │  CDC  │  Snowflake      │
│  Postgres       │──────>│  (NiFi on SPCS)       │──────>│  Tables         │
│  (source)       │       │                       │       │  (destination)  │
└─────────────────┘       └──────────────────────┘       └─────────────────┘
        │                          ▲
        │                          │ orchestrated by
        ▼                          │
┌─────────────────┐       ┌──────────────────────┐
│  Data Generator │       │  GitHub Actions       │
│  Flow (NiFi)    │       │  (nifihub CD)         │
└─────────────────┘       └──────────────────────┘
```

## Key Concepts

### OpenFlow
OpenFlow is Snowflake's managed Apache NiFi service, running inside Snowpark Container Services (SPCS). It provides data integration capabilities (connectors, flows, transformations) without managing NiFi infrastructure yourself. You interact with it through SQL commands (`CREATE OPENFLOW RUNTIME`, `SHOW OPENFLOW CONNECTORS`, etc.) and a REST API.

### NiFiHub
NiFiHub is the GitOps layer on top of OpenFlow. It stores your desired environment configuration in a YAML file (`environments/<env>/config.yaml`) and uses a GitHub Actions pipeline to reconcile the live state to match. Think of it as Terraform for OpenFlow — declarative, diff-based, and idempotent.

### CDC (Change Data Capture)
Instead of batch-copying entire tables, CDC reads the Postgres Write-Ahead Log (WAL) to capture individual inserts, updates, and deletes as they happen. This gives you near-real-time replication with minimal load on the source database.

### Snowflake Postgres
A fully managed PostgreSQL instance running inside your Snowflake account. It's accessible via standard PostgreSQL protocols (JDBC, psql) but managed through Snowflake SQL commands (`CREATE POSTGRES INSTANCE`, `SHOW POSTGRES INSTANCES`, etc.).

## Infrastructure and Cost

| Component | Where it runs | Who pays |
|---|---|---|
| CD Pipeline (detect, diff, apply) | GitHub Actions (ubuntu-latest VMs) | GitHub free tier (public repo) |
| OpenFlow Runtime (NiFi engine) | Snowflake SPCS — MEDIUM node | Snowflake credits |
| CDC Connector | Inside the OpenFlow Runtime | Snowflake credits (same node) |
| Data Generator Flow | Inside the OpenFlow Runtime | Snowflake credits (same node) |
| Snowflake Postgres | Snowflake-managed PostgreSQL | Snowflake credits |
| Cortex Code IDE | Snowflake SPCS | Snowflake credits |

The GitHub Actions workflow is lightweight — it only runs Python scripts that make API calls (~30 seconds of compute per run). The actual data processing happens inside Snowflake on the OpenFlow Runtime node, which is the primary cost driver.

For this lab, all Snowflake costs are covered by the provisioned lab account.

## How the CD Pipeline Works

The pipeline follows a **detect → diff → apply** pattern:

1. **Detect** — On every push to `main` that touches `environments/**/config.yaml`, identify which environments changed.
2. **Describe live state** — Query Snowflake and the NiFi REST API to snapshot what's currently deployed (runtimes, connectors, flows, network rules).
3. **Diff** — Compare live state against the desired config to produce a change plan (what to create, modify, or delete).
4. **Apply** — Execute the change plan via Snowflake SQL API and NiFi REST API.

This is the same reconciliation pattern used by Kubernetes controllers and Terraform. It means:
- Running the pipeline twice with the same config is safe (idempotent)
- Only what actually changed gets modified (no unnecessary teardowns)
- Removing something from config causes it to be deleted (declarative)

## Lab Steps

1. [Access the IDE](01-access-ide.md) — Launch and orient yourself in Cortex Code
2. [Fork and Configure GitHub](02-fork-configure-github.md) — Fork nifihub, create PATs, set secrets
3. [Deploy via CD Pipeline](03-deploy-cd-pipeline.md) — Configure config.yaml, push, verify deployment
4. [Start and Verify](04-start-verify.md) — Enable flows, confirm data replication
5. [Update Connector](05-update-connector.md) — Make a configuration change and redeploy

## Authentication Flow

The pipeline uses three tokens:

- **SNOWFLAKE_PAT** — Programmatic Access Token for the Snowflake SQL API (creates runtimes, connectors, network rules)
- **NIFI_RUNTIME_PAT** — Same token, used to authenticate to the NiFi REST API running inside SPCS
- **NIFIHUB_REGISTRY_PAT** — GitHub fine-grained PAT so the NiFi flow registry client can pull flow definitions from your fork

In this lab, `SNOWFLAKE_PAT` and `NIFI_RUNTIME_PAT` are the same token. In production, you'd likely use separate service accounts with scoped roles.
