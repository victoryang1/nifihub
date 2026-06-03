# Step 5: Change the Data Generator and Update the Connector

**Duration:** 15-25 minutes

## Objective
Update the data generator flow and the CDC connector configuration to include an additional table to sync into Snowflake, push to trigger the pipeline again, and verify data is flowing from Postgres into Snowflake.

**Note: This step exercises the full power of the AI agent**
    Your account is running Claude Opus 4.6, which handles complex multi-file prompts well. If using other models in your own environments, consider breaking the work into smaller steps.

## Task 1: Agent to update the flow

Using the session started earlier you can use a prompt like:

> Lab steps 1 to 4 should be complete. I now want to update the data generator flow definition to create a new table product_reviews with data being added to it. Please use plan mode to design the approach before making changes. The table should be populated with dummy data, and then added to the existing publication for replication by altering it using psql. Make sure to also update the yaml file (/home/coder/workspace/nifihub/flows/data-generator/tests/test_postgres_cdc_demo.yaml) - this is a YAML file that describes how to deploy an ephemeral runtime for testing the data generator flow. Make sure to update this YAML file to match what is specified in the demo config.yaml file so that the ephemeral runtime is properly deployed in my account. Also update the python tests for the flow where appropriate. Also update the config.yaml file of the demo account so that the connector's configuration also includes this new table.

The agent will switch to plan mode and you can give the go to build the changes and switch back to agent mode when you are happy with the plan. You will have to approve commands while the plan is being executed.

## Task 2: Push to Trigger the Pipeline

```bash
git checkout -b product_reviews
git add -A
git commit -m "add product_reviews table"
git push --set-upstream origin product_reviews
gh pr create --title "Add product_reviews table" --body "Add product_reviews table"
```

You can then check that everything is OK on the GitHub PR. 

You can then add a comment "deploy this flow" to the PR to execute the testing against an ephemeral runtime.

Once the results have been posted as a comment to the PR, you can merge the PR.

## Task 3: Verify Data is Flowing

Wait **2-3 minutes** after the pipeline completes. It can take up to 10 minutes to deploy everything.

In the SQL playground you can now check that you do have product reviews.

### Query the replicated data

```sql
SELECT * FROM DEMO_DATABASE.DEMO_POSTGRES.PRODUCT_REVIEWS LIMIT 10;
```

```sql
SELECT COUNT(*) as total_reviews FROM DEMO_DATABASE.DEMO_POSTGRES.PRODUCT_REVIEWS;
```

Run the count query again after a minute to see numbers increase (near-real-time replication).

## Validation Checklist

- [ ] Agent created the flow changes and updated config.yaml
- [ ] Changes pushed on a branch, PR created
- [ ] "deploy this flow" comment triggered ephemeral runtime test
- [ ] PR merged after test passed
- [ ] Environment CD pipeline completed successfully
- [ ] `PRODUCT_REVIEWS` table appears in `DEMO_DATABASE.DEMO_POSTGRES`
- [ ] Row count increases over time (CDC is live)
