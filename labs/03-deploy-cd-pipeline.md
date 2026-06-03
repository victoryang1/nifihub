# Step 3: Deploy via CD Pipeline

**Duration:** 15 minutes

## Objective
Configure the environment using the provided template, push to trigger the Environment CD pipeline, and verify the deployment.

## Task 1: Copy the Template

The lab folder contains a `template.yaml` with the full deployment configuration. Copy it into your nifihub fork:

```bash
cd /home/coder/workspace/nifihub
mkdir -p environments/demo
cp /home/coder/lab/template.yaml environments/demo/config.yaml
```

## Task 2: Customise the Config

You need to update the template with your specific values. The following placeholders need replacing:

### Postgres hostname and password
The Postgres password is pre-provisioned and available at `/etc/secrets/postgres_pwd/secret_string`. The CD pipeline reads it from the Snowflake secret (`DEMO_DATABASE.POSTGRES.POSTGRES_PWD`) — you don't need to paste it anywhere in config.yaml.

If you need it for direct `psql` access:
```bash
export PG_PW="$(cat /etc/secrets/postgres_pwd/secret_string)"
echo $PG_PW
```
If the Postgres password is like 'placeholder_pending_reset', then it needs to be reset

To reset the Postgres password:
```sql
ALTER POSTGRES INSTANCE HOL_POSTGRES RESET ACCESS FOR 'snowflake_admin';
ALTER SECRET DEMO_DATABASE.POSTGRES.POSTGRES_PWD SET SECRET_STRING = '<new_password>';
```

Get your Postgres hostname:
```sql
SHOW POSTGRES INSTANCES;
```

Copy the `host` value and replace it in the config (appears in 2 places — the network rule and the connector JDBC URL):

```bash
PG_HOST="<paste_your_postgres_host_here>"
sed -i "s/your-postgres-service-id.your-account.region.aws.postgres.snowflake.app/$PG_HOST/g" environments/demo/config.yaml
```

### GitHub repository owner
Replace with your GitHub username (the fork owner):
```bash
GH_USER="$(gh api user --jq .login)"
sed -i "s/Repository Owner: Snowflake-Labs/Repository Owner: $GH_USER/g" environments/demo/config.yaml
```

### Postgres schema and database names
The template has placeholder values for database, schema, user, and tables. Update them for this lab:

```bash
sed -i 's/Database Name: "mydb"/Database Name: "postgres"/g' environments/demo/config.yaml
sed -i 's/Database User: "my_user"/Database User: "snowflake_admin"/g' environments/demo/config.yaml
sed -i 's/Schema Name: "my_schema"/Schema Name: "demo_postgres"/g' environments/demo/config.yaml
sed -i 's/my_publication/demo_publication/g' environments/demo/config.yaml
sed -i 's/my_schema.table1,my_schema.table2/demo_postgres.customers,demo_postgres.orders,demo_postgres.order_items/g' environments/demo/config.yaml
sed -i 's/MY_DATABASE.MY_SCHEMA.POSTGRES_PWD/DEMO_DATABASE.POSTGRES.POSTGRES_PWD/g' environments/demo/config.yaml
sed -i 's/Snowflake Destination Database: "MY_DATABASE"/Snowflake Destination Database: "DEMO_DATABASE"/g' environments/demo/config.yaml
sed -i "s|mydb?sslmode=require|postgres?sslmode=require|g" environments/demo/config.yaml
```

### Important notes:
- Do NOT add Snowflake Connection Service or Snowflake Secret Parameter Provider definitions — these are already provisioned by the runtime
- The `provided_parameter_contexts` should match the schema where secrets live (`".*POSTGRES"` matches `DEMO_DATABASE.POSTGRES`)
- The Postgres user for this lab is `snowflake_admin`
- The warehouse is `DEFAULT_WH` (already set in the template)

### Verify your config:
```bash
grep -c "your-postgres-service-id\|MY_DATABASE\|my_schema\|my_user" environments/demo/config.yaml
```
Should return `0` — no placeholders remaining.

## Task 3: Create the POSTGRES_URL Secret

The data generator flow references `#{POSTGRES_URL}` which must be a JDBC connection string stored as a Snowflake secret in the same schema as `POSTGRES_PWD`.

Run this in the SQL Playground (replace `<pg_host>` with the host from Task 1):
```sql
CREATE SECRET IF NOT EXISTS DEMO_DATABASE.POSTGRES.POSTGRES_URL
  TYPE = GENERIC_STRING
  SECRET_STRING = 'jdbc:postgresql://<pg_host>:5432/postgres?sslmode=require';
GRANT READ ON SECRET DEMO_DATABASE.POSTGRES.POSTGRES_URL TO ROLE OPENFLOW_RUNTIME_ROLE;
```

## Task 4: Push to Trigger the Pipeline

```bash
git checkout -b demo-environment
git add -A
git commit -m "configure demo environment for my account"
git push -u origin demo-environment
```

Open a PR:
```bash
gh pr create --title "Configure demo environment" --body "Initial environment configuration for Postgres CDC lab"
```

Here we suggest you pause, and go to review the PR in Github.

**⚠️ Before merging:** Verify that GitHub Actions is active on your fork. Go to `https://github.com/<your-username>/nifihub/actions` — if you see a banner asking you to enable workflows, click it now. Without this, merging will not trigger the pipeline and you'll see "no runs found" with no error message.

Merge and watch:
```bash
gh pr merge --merge --delete-branch
gh run list --workflow "Environment CD" --limit 1
gh run watch
```

**What the pipeline does:**
1. Creates the OpenFlow Runtime with a MEDIUM node
2. Creates an External Access Integration with egress rules for Postgres and GitHub
3. Configures the flow registry client pointing to your fork
4. Sets up a Snowflake Parameter Provider reading secrets from the OPENFLOW schema
5. Deploys the data generator flow
6. Creates the CDC connector
7. Uploads the PostgreSQL JDBC driver as an asset

Pipeline typically completes in **4-5 minutes**.

## Task 5: Verify Deployment

```sql
SHOW OPENFLOW RUNTIMES IN SCHEMA OPENFLOW.OPENFLOW;
-- Expect: MY_POSTGRES_RUNTIME, status ACTIVE

SHOW OPENFLOW CONNECTORS IN SCHEMA OPENFLOW.OPENFLOW;
-- Expect: MY_POSTGRES_CONNECTOR, status STOPPED (start: false in config — started in Step 4)
```

## Troubleshooting

**Pipeline fails with "Role not granted"**
- Verify SNOWFLAKE_ROLE variable is `ACCOUNTADMIN`

**Pipeline fails with "UnknownHostException"**
- Postgres hostname not substituted: check config.yaml for placeholder text

**Pipeline fails at "Apply changes"**
- Check secrets are set: `gh secret list --env demo`
- Confirm GitHub Actions is enabled on the fork

## Validation Checklist

- [ ] `environments/demo/config.yaml` has no placeholder values remaining
- [ ] Changes pushed via PR and merged to main
- [ ] Environment CD pipeline completed successfully (green check)
- [ ] Runtime is ACTIVE
- [ ] Connector is STOPPED (started in Step 4)
