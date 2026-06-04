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

## Web IDE (SPCS) vs. Cortex Code CLI (Desktop)

This lab uses the web-based Cortex Code IDE running inside Snowpark Container Services. Here's how it compares to running Cortex Code as a native desktop app:

| | Web IDE (SPCS) | Cortex Code CLI (Desktop) |
|---|---|---|
| **Setup** | Zero — browser only, all tools pre-installed | Install app, configure Snowflake connection |
| **Pre-provisioned secrets** | Mounted at `/etc/secrets/` in container | Must export manually or use `cortex secret store` |
| **Network to Postgres** | Guaranteed (same Snowflake infra) | Depends on local network/VPN |
| **Snowflake SQL** | Built-in (same) | Built-in (same) |
| **File editing, git, gh** | Same agent capabilities | Same agent capabilities |
| **Survives laptop close** | Yes (cloud process) | No (local process) |
| **Editor experience** | Limited (REH web implementation) | Full native VS Code |
| **Extensions/plugins** | Restricted subset | Full marketplace |
| **Multi-monitor/keyboard** | Browser tab | Native OS integration |
| **Offline work** | Not possible | Possible (without Snowflake) |

**When to use the web IDE:**
- Lab/workshop settings where uniform environments matter
- Quick access from any machine without local setup
- When network access to Snowflake-internal resources (like Postgres) is required

**When to use the desktop CLI:**
- Day-to-day development with full editor features
- Projects that span Snowflake + local code/repos
- When you need native OS integration (keybindings, multi-monitor, extensions)

The agent capabilities (reading files, running SQL, executing bash, editing code) are identical in both environments.

## Lessons Learned / Gotchas

### GitHub Actions not triggering on fork
After forking `Snowflake-Labs/nifihub`, workflows are listed as "active" but won't actually fire until you click **"I understand my workflows, go ahead and enable them"** on the Actions tab. There's no error — pushes just silently produce zero runs. Always verify with `gh run list` after your first push.

### Snowflake PAT expiry
The pre-provisioned PAT at `/etc/secrets/pat/secret_string` may be invalid by the time you use it. The pipeline will fail with `Programmatic access token is invalid`. Fix: regenerate with `ALTER USER "USER" ADD PROGRAMMATIC ACCESS TOKEN ...` and update **both** `SNOWFLAKE_PAT` and `NIFI_RUNTIME_PAT` GitHub secrets (they're the same token in this lab).

### CDC connectors don't hot-reload table lists
Changing `Included Comma Separated Source Table Names` in `config.yaml` is not a live parameter tweak — it requires a **new connector version**. The sequence:

1. Add the table to the Postgres publication (`ALTER PUBLICATION ... ADD TABLE`)
2. Update `config.yaml` with the new table in the connector's `Included` list
3. Push to trigger the CD pipeline
4. The pipeline must detect the drift and create a new connector version
5. The connector restarts with the new version and does a fresh snapshot of the new table

If the pipeline runs too fast (detects "no changes" because the connector was still running with the old version), you may need to push again. The diff engine compares live state vs. desired — if the connector was mid-transition (STOPPING), the diff may not detect the mismatch on the first pass.

**Key insight:** Adding a table to the publication alone is not enough. The connector itself must be re-deployed with a config that includes the table. Think of it as a schema migration, not a runtime setting.

### Pipeline ran successfully but nothing happened
A green pipeline doesn't always mean changes were applied. The diff engine may have determined "no changes needed" if:
- The connector was already running and its live parameters matched (or the diff couldn't detect the mismatch)
- The flow version in the registry hadn't updated yet

Check the "Change Plan" step output in GitHub Actions — if `changes.json` is empty, the pipeline was a no-op.

### Connector stop/start vs. re-deploy
- **Stop/Start** — Restarts the connector with its current version. Does NOT pick up new config.
- **New version (re-deploy)** — The CD pipeline creates a new connector version with updated parameters. This is what's needed for table list changes.

Manual stop/start via `ALTER OPENFLOW CONNECTOR ... STOP/START` is useful for debugging, but won't fix config drift.

### Flow registry and version: latest
The flow registry client pulls from your GitHub fork. `version: latest` means "latest commit on main in the flows path." After merging a PR that changes the flow JSON, the registry sees the new version on the next deploy. But if the pipeline runs before GitHub's API has propagated the merge, it may still see the old version.

### Connector update lifecycle (what the pipeline actually does)
When the diff detects a connector parameter change, `orchestrate.py` calls into `manage_connectors.py` which runs this sequence:

1. `stop_connector()` — `ALTER OPENFLOW CONNECTOR ... STOP`, waits for STOPPED
2. `get_connector_config()` — Downloads the live `config.json` from the connector's internal stage
3. `apply_connector_config()` — Patches the JSON with new parameter values
4. `put_connector_config()` — Uploads the modified config back to the stage
5. `add_live_version()` — `ALTER OPENFLOW CONNECTOR ... ADD LIVE VERSION FROM LAST`
6. `commit_connector()` — `ALTER OPENFLOW CONNECTOR ... COMMIT`
7. `start_connector()` — `ALTER OPENFLOW CONNECTOR ... START`, waits for RUNNING

This is all automated inside the pipeline — no manual intervention needed. The full cycle takes 2-4 minutes due to the stop/start wait times.

### Network rule port requirement
The Postgres network rule in `config.yaml` **must** include the port (`:5432`). Just the hostname without a port won't work for SPCS egress rules.

### The `demo` GitHub environment must exist before setting secrets
GitHub doesn't auto-create environments. If you run `gh secret set --env demo` before the environment exists, you get a 404. Create it first: `gh api --method PUT "repos/<user>/nifihub/environments/demo"`

### Whitespace pushes to retrigger
If you need to retrigger the Environment CD pipeline without real changes, append a newline to `config.yaml`. The workflow only fires on changes to `environments/**/config.yaml`, so changing other files won't trigger it. Empty commits won't work either (no path match in the diff).
