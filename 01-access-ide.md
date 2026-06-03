# Step 1: Access the Cortex Code IDE

**Duration:** 5 minutes

## Objective
Log in to Snowsight, launch the Cortex Code IDE service, and orient yourself in the environment.

## Task 1: Log In to Snowsight

Open a browser and navigate to https://app.snowflake.com. Log in with the credentials provided at the event.

## Task 2: Launch the IDE

1. In Snowsight, navigate to **Monitoring** → **Services & Jobs**
2. Find **COCO_DESKTOP_SERVICE** in the list
3. If the service is **Suspended**, click the **⋯** (three dots) menu on the right and select **Resume**. Wait for it to become **Running** (1-2 minutes)
4. Click into the service to open its details
5. Under **Endpoints**, find **ide-ui** and click the **↗** arrow icon (or copy the URL and open it in a new tab)

**Note:** The IDE will auto-suspend after 4 hours of inactivity (no Snowflake queries). If this happens, just go back to Services & Jobs and Resume it. Your workspace files are preserved — only conversation history may be lost.

## Task 3: Orient Yourself

This is an experimental REH Web implementation of Cortex Code Desktop. 
It does not have the full functionality of the native Windows/OSX/Linux application, but most things should work as expected - please let us know your feedback.

The IDE has two modes — **Agent** and **Editor** — you can switch between them using the buttons in the top-right corner or with **Cmd+E** (Mac) / **Ctrl+E** (Windows).

Switch to **Editor** mode and explore:

### Open the Workspace
1. In the file browser pane, open the folder `/home/coder`

You should now see both `workspace` and `lab` in the file browser. The `lab` folder contains step-by-step guides and agent instructions (read-only).

### Check the Snowflake Catalog
Click the **Snowflake icon** in the sidebar (Snowflake Catalog) — it shows the databases in your Snowflake account. You should see DEMO_DATABASE, OPENFLOW, and others.

### Test the Agent
Switch to **Agent** mode and ask:

> Read /home/coder/lab/AGENTS.md and the instructions files, and summarise what this lab is about for me

This confirms the agent is connected and can read files.

### Check the Terminal
Open a terminal (**Cmd+J** on Mac, or **Ctrl+`**) and verify environment variables are populated:

```bash
echo "Host: $SNOWFLAKE_HOST"
echo "User: $SNOWFLAKE_ACCOUNT"
```

Also confirm the key tools are available:
```bash
gh --version
git --version
```

## Validation Checklist

- [ ] Logged in to Snowsight
- [ ] COCO_DESKTOP_SERVICE is Running
- [ ] IDE loaded in a browser tab via the ide-ui endpoint
- [ ] Can switch between Agent and Editor modes (Cmd+E)
- [ ] Workspace folder open, lab folder added
- [ ] Database browser shows account databases
- [ ] Agent responds and can read lab files
- [ ] Terminal shows environment variables populated
