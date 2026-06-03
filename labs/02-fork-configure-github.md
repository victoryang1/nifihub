# Step 2: Fork, Clone, and Configure GitHub

**Duration:** 15 minutes

## Objective
Fork the nifihub repository, create a scoped GitHub PAT, clone into the IDE, and configure the GitHub Actions secrets and variables.

## Task 1: Fork the Nifihub Repository

1. Go to https://github.com/Snowflake-Labs/nifihub
2. Click **Fork** → choose your personal GitHub account
3. Keep default name (`nifihub`) → Click **Create fork**
4. **Immediately after forking**, go to your fork's **Actions** tab and click **"I understand my workflows, go ahead and enable them"**
5. Then go to **Settings** → **Environments** → **New environment** → name: `demo` → **Configure environment** → Save

Both steps 4 and 5 are required before the CD pipeline can run.

## Task 2: Create a Fine-Grained GitHub PAT

Go to: https://github.com/settings/personal-access-tokens/new

| Setting | Value |
|---------|-------|
| Token name | `nifihub-hol` |
| Expiration | 30 days |
| Repository access | Only select repositories → your fork |

**Permissions → Repository permissions:**
| Permission | Access |
|-----------|--------|
| Actions | Read and write |
| Administration | Read and write |
| Contents | Read and write |
| Environments | Read and write |
| Metadata | Read-only |
| Pull requests | Read and write |
| Secrets | Read and write |
| Variables | Read and write |
| Workflows | Read and write |

Generate and copy the token.

## Task 3: Clone in the IDE Terminal

```bash
export GH_PAT="<paste_your_pat_here>"
echo "$GH_PAT" | gh auth login --with-token
```

Clone your fork:
```bash
cd /home/coder/workspace
git clone https://${GH_PAT}@github.com/<your_username>/nifihub.git
cd nifihub
```

Configure git identity:
```bash
git config --global user.email "attendee@summit.snowflake.com"
git config --global user.name "HOL Attendee"
```

Configure git push authentication (gh auth doesn't cover git push):
```bash
export GH_USER="<your_github_username>"
git remote set-url origin "https://x-access-token:${GH_PAT}@github.com/${GH_USER}/nifihub.git"
```

Set default repo for gh CLI (needed for `gh pr create`):
```bash
gh repo set-default ${GH_USER}/nifihub
```

**Do NOT use `gh auth login --web`** — enterprise policies may block the broad OAuth scopes.

## Task 4: Get the Snowflake PAT

A Snowflake PAT (for user `USER` with role `ACCOUNTADMIN`) has been pre-provisioned and is available at:

```bash
export SNOWFLAKE_PAT="$(cat /etc/secrets/pat/secret_string)"
echo "Snowflake PAT loaded: ${SNOWFLAKE_PAT:0:20}..."
```

This is the **Snowflake** PAT used for the CD pipeline to authenticate to Snowflake — it is separate from the GitHub PAT you created in Task 2.

If you need to reset it (e.g. it has expired), run in the SQL Playground:
```sql
ALTER USER "USER" ADD PROGRAMMATIC ACCESS TOKEN NIFIHUB_CD_PAT
  ROLE_RESTRICTION = 'ACCOUNTADMIN' DAYS_TO_EXPIRY = 90;
```
Then copy the new value:
```bash
export SNOWFLAKE_PAT="<paste_new_pat_here>"
```

**Important:** If you regenerate the PAT, update BOTH `SNOWFLAKE_PAT` and `NIFI_RUNTIME_PAT` GitHub secrets with the new value — they use the same token in this lab.

## Task 5: Set GitHub Secrets

```bash
gh secret set SNOWFLAKE_PAT --env demo --body "$SNOWFLAKE_PAT"
gh secret set NIFI_RUNTIME_PAT --env demo --body "$SNOWFLAKE_PAT"
gh secret set NIFIHUB_REGISTRY_PAT --env demo --body "$GH_PAT"
```

| Secret | Purpose |
|--------|---------|
| SNOWFLAKE_PAT | CD pipeline authenticates to Snowflake SQL API |
| NIFI_RUNTIME_PAT | CD pipeline authenticates to the OpenFlow Runtime NiFi API |
| NIFIHUB_REGISTRY_PAT | Flow registry client authenticates to GitHub |

## Task 6: Set GitHub Variables

```bash
gh variable set SNOWFLAKE_ACCOUNT_URL --env demo --body "https://${SNOWFLAKE_HOST}"
gh variable set SNOWFLAKE_USER --env demo --body "USER"
gh variable set SNOWFLAKE_ROLE --env demo --body "ACCOUNTADMIN"
```

## Validation Checklist

- [ ] Fork of nifihub exists in your GitHub account
- [ ] GitHub Actions enabled on the fork
- [ ] Fine-grained PAT created and saved
- [ ] `gh auth status` shows authenticated
- [ ] Repo cloned at `/home/coder/workspace/nifihub`
- [ ] All 3 secrets set (`gh secret list --env demo`)
- [ ] All 3 variables set (`gh variable list --env demo`)
