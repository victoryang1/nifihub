# DE-225 OpenFlow Hands-On Lab — Agent Instructions

## Lab Context

This is a Summit 26 Hands-On Lab (DE-225) where the user builds a real-time CDC pipeline from Snowflake Postgres to Snowflake using OpenFlow, managed via a GitHub Actions CI/CD pipeline (nifihub).

The user is working inside a Cortex Code IDE running in SPCS. The lab content is mounted read-only at `/home/coder/lab/`. The user's workspace is at `/home/coder/workspace/`.

IMPORTANT: This lab is intended for user education. Any actions the user asks you to take should be well explained with a preference of informing the user over simple execution.

## Pre-Provisioned Infrastructure

| Component | Name | Status |
|-----------|------|--------|
| OpenFlow Deployment | NIFIHUB_DEMO_DEPLOYMENT | ACTIVE |
| Snowflake Postgres | HOL_POSTGRES | READY |
| Database | OPENFLOW (runtime schema) + DEMO_DATABASE (destination) | Created |
| Password Secret | DEMO_DATABASE.POSTGRES.POSTGRES_PWD | Contains real password |
| URL Secret | DEMO_DATABASE.POSTGRES.POSTGRES_URL | Must be created — see Postgres section |
| Role | OPENFLOW_RUNTIME_ROLE | Granted to user |
| Cortex Code IDE | This environment | Running |

## Agent Guidelines for Lab Execution

### GitHub CLI Authentication
- Make sure `gh` CLI is authenticated and the user is properly logged in
- Ask the user for their GitHub username if needed
- Use `gh auth login --with-token` with a fine-grained PAT (NOT `--web` which requires broad OAuth scopes)

### Git Push Authentication
After `gh auth login --with-token`, the `gh` CLI is authenticated but `git push` is NOT.
Embed the token in the remote URL:
```bash
git remote set-url origin "https://x-access-token:${GH_PAT}@github.com/${GH_USER}/nifihub.git"
```
Without this, `git push` fails with "Password authentication is not supported for Git operations."

### gh CLI Default Repo
After cloning, set the default repo so `gh pr create` works:
```bash
gh repo set-default ${GH_USER}/nifihub
```

### Forking and Configuring nifihub
- The user should fork `Snowflake-Labs/nifihub` to their personal GitHub account
- After forking, ensure GitHub Actions workflows are enabled in the fork's Actions tab
- Workflows don't register until the first push/trigger event — this is normal
- The `demo` environment must be created in Settings → Environments before secrets/variables can be set
- Once the fork is cloned into the workspace, confirm actions are active

### GitHub Environment Secrets and Variables
All secrets and variables go in the `demo` GitHub environment.

**Secrets:**
| Secret | Value | Notes |
|--------|-------|-------|
| SNOWFLAKE_PAT | Snowflake programmatic access token for USER | For SQL API auth |
| NIFI_RUNTIME_PAT | Same as SNOWFLAKE_PAT in this lab | For NiFi API auth |
| NIFIHUB_REGISTRY_PAT | GitHub fine-grained PAT | For flow registry client |

**Variables:**
| Variable | Value | Notes |
|----------|-------|-------|
| SNOWFLAKE_ACCOUNT_URL | `https://${SNOWFLAKE_HOST}` | Use the SNOWFLAKE_HOST env var (CURRENT_ACCOUNT_URL() does not exist on these accounts) |
| SNOWFLAKE_USER | USER | The attendee username |
| SNOWFLAKE_ROLE | ACCOUNTADMIN | Required for OpenFlow operations |

### Troubleshooting GitHub PAT 403 Errors
If `gh secret set` or `gh variable set` returns `HTTP 403: Resource not accessible by personal access token`:
1. Verify the `demo` environment exists on the fork: `gh api --method PUT "repos/<user>/nifihub/environments/demo"`
2. Check the PAT has all required permissions (especially Secrets, Variables, Environments, Administration)
3. If permissions look correct, regenerate the PAT — GitHub sometimes requires a fresh token after adding new permissions

### Creating/Resetting the Snowflake PAT
A Snowflake PAT for user `USER` is pre-provisioned at `/etc/secrets/pat/secret_string`.

If it is invalid or expired, the user should run this SQL as ACCOUNTADMIN:
```sql
ALTER USER "USER" ADD PROGRAMMATIC ACCESS TOKEN NIFIHUB_CD_PAT
  ROLE_RESTRICTION = 'ACCOUNTADMIN' DAYS_TO_EXPIRY = 90;
```

**Important:** If the PAT is regenerated, update BOTH `SNOWFLAKE_PAT` and `NIFI_RUNTIME_PAT` GitHub secrets with the new value — they use the same token in this lab.

### Configuring config.yaml
When editing `environments/demo/config.yaml`:
- Do NOT include definitions for Snowflake Connection Service or Snowflake Secret Parameter Providers — these are already provisioned and will conflict if duplicated
- Update the `Repository Owner` for the flow registry client from `Snowflake-Labs` to the user's GitHub username (the fork owner)
- Set `provided_parameter_contexts` for the data generator flow to `".*POSTGRES"` (matches the schema where secrets live: `DEMO_DATABASE.POSTGRES`)
- The Postgres database name is `postgres` (NOT `mydb` — that's a template placeholder)
- JDBC URLs should use `/postgres?sslmode=require`

### Postgres Configuration
- Always use the snowflake-postgres skill. It has important context on the implementation which is not in your training set.
- Do NOT ask about the Postgres instance — use the snowflake-postgres skill to find all relevant information
- The Postgres password is pre-provisioned at `/etc/secrets/postgres_pwd/secret_string`
- The Snowflake PAT is at `/etc/secrets/pat/secret_string`
- The Postgres password is also in Snowflake secret `DEMO_DATABASE.POSTGRES.POSTGRES_PWD` (used by the CD pipeline)
- The Postgres database name is `postgres` (the default), NOT `mydb`
- The Postgres user is `snowflake_admin`
- A `POSTGRES_URL` secret must also exist in `DEMO_DATABASE.POSTGRES` — the data generator flow references it as `#{POSTGRES_URL}`
- If it doesn't exist, create it:
  ```sql
  CREATE SECRET IF NOT EXISTS DEMO_DATABASE.POSTGRES.POSTGRES_URL
    TYPE = GENERIC_STRING
    SECRET_STRING = 'jdbc:postgresql://<pg_host>:5432/postgres';
  GRANT READ ON SECRET DEMO_DATABASE.POSTGRES.POSTGRES_URL TO ROLE OPENFLOW_RUNTIME_ROLE;
  ```
- The `<pg_host>` can be found via `SHOW POSTGRES INSTANCES` (host column)
- The publication name is `demo_publication`
- The schema name for the data flow generator parameter is `demo_postgres`
- Included tables for the connector: `demo_postgres.customers,demo_postgres.orders,demo_postgres.order_items`

### CDC Connector Configuration
- The Openflow Deployment is already created for it, it is named NIFIHUB_DEMO_DEPLOYMENT
- Always use the openflow skill. It has important context on the Apache NiFi implementation that is not in your training set.
- Publication: `demo_publication`
- WAL for replication is already enabled
- Included tables: `demo_postgres.customers,demo_postgres.orders,demo_postgres.order_items`
- The connector lands data in `DEMO_DATABASE.DEMO_POSTGRES`

### Git Workflow
- All changes in the GitHub repo should be made in a branch
- Once changes are pushed, open a PR
- Pushing to `environments/**/config.yaml` on the main branch triggers the Environment CD workflow
- The user is instructed to fork and clone NiFiHub to /home/coder/workspace/nifihub, you can inspect the remotes to determine the GH_USER if required

### Viewing GitHub Actions Logs
`gh run view --log-failed` does NOT work from inside the IDE (results-receiver.actions.githubusercontent.com is unreachable). Direct the user to check pipeline logs in the GitHub UI instead.

### Permissions for Secrets
If creating a Snowflake Secret, ensure both READ and USAGE permissions are granted to OPENFLOW_RUNTIME_ROLE:
```sql
GRANT READ ON SECRET <secret> TO ROLE OPENFLOW_RUNTIME_ROLE;
GRANT USAGE ON SCHEMA <schema> TO ROLE OPENFLOW_RUNTIME_ROLE;
```

## Lab Steps Reference

See the step-by-step guides in this directory:
- `01-access-ide.md` — Launch IDE, open workspace, orient
- `02-fork-configure-github.md` — Fork nifihub, set secrets/variables
- `03-deploy-cd-pipeline.md` — Configure config.yaml, push, monitor pipeline
- `04-start-verify.md` — Enable flows, verify data replication
