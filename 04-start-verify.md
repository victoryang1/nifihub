# Step 4: Start and Verify Data Flow

**Duration:** 10 minutes

## Objective
Enable the data generator flow and CDC connector, push to trigger the pipeline again, and verify data is flowing from Postgres into Snowflake.

## Task 1: Enable Auto-Start

Edit `environments/demo/config.yaml` to start the flow and connector:

```bash
cd /home/coder/workspace/nifihub
sed -i 's/start: false/start: true/g' environments/demo/config.yaml
```

This changes both the data generator flow and the CDC connector from `start: false` to `start: true`.

## Task 2: Push to Trigger the Pipeline

```bash
git add -A
git commit -m "enable auto-start for flows and connector"
git push
```

The pipeline will:
1. Detect the flow and connector already exist
2. Start the data generator flow
3. Start the CDC connector

Watch:
```bash
gh run watch
```

## Task 3: Verify Data is Flowing

Wait **2-3 minutes** after the pipeline completes.

In the Snowflake extension:

```sql
SELECT table_name, row_count
FROM OPENFLOW.INFORMATION_SCHEMA.TABLES
WHERE table_schema = 'DEMO_POSTGRES';
```

You should see `CUSTOMERS`, `ORDERS`, and `ORDER_ITEMS` with growing row counts.

### Query the replicated data

```sql
SELECT * FROM DEMO_DATABASE.DEMO_POSTGRES.CUSTOMERS LIMIT 10;
```

```sql
SELECT COUNT(*) as total_orders FROM DEMO_DATABASE.DEMO_POSTGRES.ORDERS;
```

Run the count query again after a minute to see numbers increase (near-real-time replication).

## Troubleshooting

**No tables appear in DEMO_POSTGRES**
- Wait 2-3 minutes — initial snapshot takes time
- Check connector status: `SHOW OPENFLOW CONNECTORS IN SCHEMA OPENFLOW.OPENFLOW` — should be RUNNING
- If FAILED, check connector event log in Snowsight

**Connector shows FAILED status**
- Ensure Postgres tables use REPLICA IDENTITY DEFAULT
- May need full reinstall: stop → terminate → drop → redeploy via pipeline

**Flow can't connect to Postgres (UnknownHostException)**
- Verify EAI network rule includes your Postgres host:
```sql
DESCRIBE NETWORK RULE OPENFLOW.OPENFLOW.POSTGRES_EGRESS;
```

## Validation Checklist

- [ ] `start: true` set for both flow and connector
- [ ] Environment CD pipeline completed successfully
- [ ] `DEMO_POSTGRES` connector is RUNNING
- [ ] Tables exist in `DEMO_DATABASE.DEMO_POSTGRES` with row counts > 0
- [ ] Count query shows increasing numbers over time

## Congratulations!

You've completed the OpenFlow Hands-On Lab:
- Accessed a browser-based Cortex Code IDE in SPCS
- Set up Snowflake Postgres as a CDC source
- Configured nifihub for GitOps-style OpenFlow management
- Deployed an OpenFlow Runtime, data generator flow, and CDC connector via CI/CD
- Verified real-time data replication from Postgres to Snowflake
