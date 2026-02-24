# Troubleshooting Guide: Database as a Tool — Agentic RAG with ADK, MCP Toolbox, and Cloud SQL

> **Codelab URL:** https://codelabs.developers.google.com/agentic-rag-toolbox-cloudsql
>
> **Last updated:** 2026-02-24

This guide helps you diagnose and fix common issues encountered while following the **Database as a Tool: Agentic RAG with ADK, MCP Toolbox, and Cloud SQL** codelab. If you're stuck, start with the [Quick Fixes](#quick-fixes) section — it covers the most frequent problems.

---

## Table of Contents

- [Quick Fixes](#quick-fixes)
- [Step-by-Step Troubleshooting](#step-by-step-troubleshooting)
  - [Step 1: Set Up Your Environment](#step-1-set-up-your-environment)
  - [Step 2: Create the Database Instance](#step-2-create-the-database-instance)
  - [Step 3: Initialize the Agent Project](#step-3-initialize-the-agent-project)
  - [Step 4: Seed the Jobs Listing Database](#step-4-seed-the-jobs-listing-database)
  - [Step 5: Configure MCP Toolbox for Databases](#step-5-configure-mcp-toolbox-for-databases)
  - [Step 6: Build the ADK Agent](#step-6-build-the-adk-agent)
  - [Step 7: Deploy to Cloud Run](#step-7-deploy-to-cloud-run)
  - [Step 8: Congratulations / Clean Up](#step-8-congratulations--clean-up)
- [General FAQ](#general-faq)
- [Environment Recovery](#environment-recovery)

---

## Quick Fixes

The most common issues across all steps. **Check these first** before diving into step-specific troubleshooting.

### 1. Environment variables are empty after switching terminal tabs

> **Symptom:** Commands like `echo $GOOGLE_CLOUD_PROJECT` or `echo $DB_PASSWORD` return blank. `gcloud` commands fail with project-related errors.
>
> **Cause:** This codelab requires opening multiple terminal tabs (Steps 2, 4, 5, 7). Each tab has its own shell session — environment variables from `.env` are only loaded in the tab where you ran `source .env`.
>
> **Fix:**
> ```bash
> cd ~/build-agent-adk-toolbox-cloudsql
> bash setup_verify_trial_project.sh && source .env
> ```
> Verify with `echo $GOOGLE_CLOUD_PROJECT` — it should print your project ID.

### 2. Cloud Shell session disconnected

> **Symptom:** Cloud Shell terminal is unresponsive, shows "Reconnecting...", or you return to a fresh terminal without your environment variables. All background processes (Cloud SQL Auth Proxy, Toolbox server, embedding generation) are gone.
>
> **Cause:** Cloud Shell timed out due to inactivity (default: 20 minutes idle) or a network interruption. All background processes are killed on disconnection.
>
> **Fix:** See the [Environment Recovery](#environment-recovery) section for full recovery steps. The short version:
> ```bash
> cd ~/build-agent-adk-toolbox-cloudsql
> bash setup_verify_trial_project.sh && source .env
> ```
> Then restart whichever background processes you need for your current step.

### 3. Port 5432 already in use (Cloud SQL Auth Proxy conflict)

> **Symptom:** `unable to start: listen tcp 127.0.0.1:5432: bind: address already in use` when starting the Cloud SQL Auth Proxy.
>
> **Cause:** A previous proxy process is still running from an earlier terminal tab or session.
>
> **Fix:**
> ```bash
> lsof -ti:5432 | xargs kill -9 2>/dev/null || fuser -k 5432/tcp 2>/dev/null
> cloud-sql-proxy ${GOOGLE_CLOUD_PROJECT}:${REGION}:jobs-instance --port 5432 &
> ```

### 4. Port 5000 already in use (Toolbox server conflict)

> **Symptom:** Toolbox fails to start with `address already in use` or similar bind error on port 5000.
>
> **Cause:** A previous Toolbox process is still running in another terminal tab.
>
> **Fix:**
> ```bash
> lsof -ti:5000 | xargs kill -9 2>/dev/null || fuser -k 5000/tcp 2>/dev/null
> ```
> Then restart the Toolbox server as described in Step 5.

### 5. Port 8000 already in use (ADK dev UI conflict)

> **Symptom:** `ERROR: [Errno 98] Address already in use` or `ERROR: Could not bind to port 8000` when running `uv run adk web`.
>
> **Cause:** A previous `adk web` process wasn't stopped with Ctrl+C, or is still running in another terminal tab.
>
> **Fix:**
> ```bash
> lsof -ti:8000 | xargs kill -9 2>/dev/null || fuser -k 8000/tcp 2>/dev/null
> ```
> If both commands fail, try: `uv run adk web --port 8001` and use port 8001 in Web Preview instead.

### 6. Cloud SQL Auth Proxy not running

> **Symptom:** `psql` commands fail with `could not connect to server: Connection refused` or `Is the server running on host "127.0.0.1" and accepting TCP/IP connections on port 5432?`
>
> **Cause:** The Cloud SQL Auth Proxy background process was killed — either by Cloud Shell disconnection, closing the terminal tab where it was running, or pressing Ctrl+C in the wrong tab.
>
> **Fix:**
> ```bash
> # Check if the proxy is running
> ss -tlnp | grep ':5432 '
>
> # If not running, restart it
> cd ~/build-agent-adk-toolbox-cloudsql
> source .env
> cloud-sql-proxy ${GOOGLE_CLOUD_PROJECT}:${REGION}:jobs-instance --port 5432 &
> ```

### 7. Toolbox server not running

> **Symptom:** `curl http://localhost:5000/api/toolset` returns `Connection refused`. The ADK agent fails to load tools with connection errors.
>
> **Cause:** The Toolbox background process was killed — by Cloud Shell disconnection, closing its terminal tab, or pressing Ctrl+C.
>
> **Fix:**
> ```bash
> cd ~/build-agent-adk-toolbox-cloudsql
> source .env
> export GOOGLE_CLOUD_PROJECT=$GOOGLE_CLOUD_PROJECT
> export GOOGLE_CLOUD_LOCATION=$GOOGLE_CLOUD_LOCATION
> export GOOGLE_GENAI_USE_VERTEXAI=true
> export DB_PASSWORD=$DB_PASSWORD
> export REGION=$REGION
> ./toolbox --tools-file tools.yaml &
> ```
> Wait for `Server ready to serve!` before proceeding.

### 8. Vertex AI API errors

> **Symptom:** Embedding generation fails, Toolbox returns errors about Vertex AI, or the agent responds with `Vertex AI API has not been used in project` or `403 Permission Denied`.
>
> **Cause:** The Vertex AI API wasn't enabled in Step 1, or enablement hasn't propagated yet.
>
> **Fix:**
> ```bash
> gcloud services enable aiplatform.googleapis.com
> ```
> Wait for `Operation/... finished successfully` before retrying.

---

## Step-by-Step Troubleshooting

### Step 1: Set Up Your Environment

#### Issue: Trial billing account redemption fails

> **Symptom:** The redemption portal shows an error, says the code is invalid, or the page doesn't load.
>
> **Cause:** You may be using a Google Workspace email instead of a personal Gmail, or the redemption code has expired.
>
> **Fix:**
> 1. Open an **incognito window** and log in with a **personal Gmail** account (not Workspace/corporate email).
> 2. If the code is expired, ask your workshop instructor for a new one.
> 3. If self-studying, create a project with your own billing account instead.

#### Issue: `cloudshell workspace` command doesn't change the editor view

> **Symptom:** After running `cloudshell workspace ~/build-agent-adk-toolbox-cloudsql && cd ~/build-agent-adk-toolbox-cloudsql`, the Cloud Shell Editor file tree still shows a different directory.
>
> **Cause:** The `cloudshell workspace` command sometimes requires a page refresh to take effect in the editor.
>
> **Fix:**
> 1. Refresh the Cloud Shell browser tab.
> 2. If the editor still shows the wrong directory, click **File > Open Folder** in the Cloud Shell Editor and navigate to `~/build-agent-adk-toolbox-cloudsql`.
> 3. The terminal `cd` command works independently — verify with `pwd`.

#### Issue: Setup script fails with "no trial billing account found"

> **Symptom:** `setup_verify_trial_project.sh` exits with an error about no trial billing account.
>
> **Cause:** The trial billing account was not successfully linked in the redemption step, or you're logged into Cloud Shell with a different Google account than the one you redeemed with.
>
> **Fix:**
> ```bash
> # Check which account is active
> gcloud auth list
> ```
> If the active account is wrong:
> ```bash
> gcloud auth login
> ```
> Then re-run the script: `bash setup_verify_trial_project.sh && source .env`

#### Issue: API enablement fails

> **Symptom:** `ERROR: (gcloud.services.enable) FAILED_PRECONDITION: Billing account for project is not found.`
>
> **Cause:** No billing account is linked to the project.
>
> **Fix:**
> 1. Go to [Billing](https://console.cloud.google.com/billing/linkedaccount).
> 2. Link your trial billing account.
> 3. Retry:
> ```bash
> gcloud services enable \
>   aiplatform.googleapis.com \
>   sqladmin.googleapis.com \
>   compute.googleapis.com \
>   run.googleapis.com \
>   cloudbuild.googleapis.com \
>   artifactregistry.googleapis.com
> ```

#### Issue: Terminal shows `(base)` prefix or conda interference

> **Symptom:** Terminal prompt shows `(base)` and Python commands use the conda-managed Python instead of `uv`.
>
> **Cause:** Cloud Shell may have conda pre-activated in some configurations.
>
> **Fix:**
> ```bash
> conda deactivate
> ```
> This won't affect the codelab — `uv` manages its own virtual environment independently.

---

### Step 2: Create the Database Instance

#### Issue: Cloud SQL instance creation fails immediately

> **Symptom:** The `gcloud sql instances create` command returns an error about quotas, permissions, or the Compute Engine API not being enabled.
>
> **Cause:** The Compute Engine API or Cloud SQL Admin API wasn't enabled in Step 1, or API enablement hasn't propagated yet.
>
> **Fix:**
> ```bash
> # Ensure required APIs are enabled
> gcloud services enable sqladmin.googleapis.com compute.googleapis.com
>
> # Wait a moment, then retry instance creation
> source .env
> gcloud sql instances create jobs-instance \
>   --database-version=POSTGRES_17 \
>   --tier=db-custom-1-3840 \
>   --edition=ENTERPRISE \
>   --region=$REGION \
>   --root-password=$DB_PASSWORD \
>   --enable-google-ml-integration \
>   --database-flags cloudsql.enable_google_ml_integration=on \
>   --quiet &
> ```

#### Issue: Instance already exists

> **Symptom:** `ERROR: (gcloud.sql.instances.create) Resource already exists` when running the create command.
>
> **Cause:** You ran the creation command twice, or a previous attempt completed in the background.
>
> **Fix:** Check the instance status:
> ```bash
> gcloud sql instances describe jobs-instance --format="value(state)"
> ```
> If it returns `RUNNABLE`, the instance is ready — skip to Step 3. If it returns `PENDING_CREATE`, wait for it to finish. If it shows an error state:
> ```bash
> gcloud sql instances delete jobs-instance --quiet
> # Then re-run the creation command from the codelab
> ```

#### Issue: Background process output floods the terminal

> **Symptom:** While typing in the terminal, Cloud SQL creation progress messages or the Toolbox download progress appear and disrupt your input.
>
> **Cause:** The `&` suffix runs the commands in the background, but their stdout/stderr still print to the current terminal.
>
> **Fix:** This is expected behavior. Open a **new terminal tab** (click the `+` icon in Cloud Shell) as instructed. Run the environment setup in the new tab:
> ```bash
> cd ~/build-agent-adk-toolbox-cloudsql
> bash setup_verify_trial_project.sh && source .env
> ```

#### Issue: Toolbox binary download hangs or fails

> **Symptom:** The `curl -O https://storage.googleapis.com/genai-toolbox/v0.27.0/linux/amd64/toolbox &` command doesn't complete, or the file is not present later.
>
> **Cause:** Network issues in Cloud Shell, or the download was interrupted by a session timeout.
>
> **Fix:**
> ```bash
> cd ~/build-agent-adk-toolbox-cloudsql
> # Remove partial download if present
> rm -f toolbox
> # Re-download (without background this time so you can see progress)
> curl -O https://storage.googleapis.com/genai-toolbox/v0.27.0/linux/amd64/toolbox
> chmod +x toolbox
> ```

#### Issue: `DB_PASSWORD` variable is empty in the new terminal tab

> **Symptom:** In the new terminal tab, `echo $DB_PASSWORD` returns blank.
>
> **Cause:** Environment variables are per-tab. The `DB_PASSWORD` was written to `.env` in the first tab, but the new tab hasn't sourced that file.
>
> **Fix:**
> ```bash
> cd ~/build-agent-adk-toolbox-cloudsql
> source .env
> echo $DB_PASSWORD  # Should print: techjobs-pwd-2025
> ```

---

### Step 3: Initialize the Agent Project

#### Issue: `uv init` fails or creates unexpected files

> **Symptom:** `uv init` fails with an error or doesn't create `pyproject.toml`.
>
> **Cause:** There's already a `pyproject.toml` in the directory from a previous attempt.
>
> **Fix:**
> ```bash
> # If retrying, remove old project files first
> rm -f pyproject.toml uv.lock
> rm -rf .venv
> uv init
> ```

#### Issue: `uv add google-adk==1.25.0 toolbox-adk==0.6.0` fails

> **Symptom:** Dependency resolution errors or network timeouts during `uv add`.
>
> **Cause:** Network issues in Cloud Shell, or a version conflict with the Python version.
>
> **Fix:**
> ```bash
> # Retry with increased timeout
> uv add google-adk==1.25.0 toolbox-adk==0.6.0 --timeout 120
> ```
> If it still fails, check your Python version: `uv run python --version`. The ADK requires Python 3.10+.

#### Issue: `adk create` fails with "command not found"

> **Symptom:** `uv run adk create` returns `error: No such command: adk`.
>
> **Cause:** The `google-adk` package wasn't installed, or `uv add` failed silently.
>
> **Fix:**
> ```bash
> # Verify the package is installed
> uv pip list | grep google-adk
>
> # If not listed, add it again
> uv add google-adk==1.25.0
> ```

#### Issue: `jobs_agent` directory already exists

> **Symptom:** `adk create jobs_agent` fails because the directory already exists from a previous attempt.
>
> **Cause:** You ran `adk create` before (possibly with an error) and the directory was partially created.
>
> **Fix:**
> ```bash
> rm -rf jobs_agent
> uv run adk create jobs_agent \
>     --model gemini-2.5-flash \
>     --project ${GOOGLE_CLOUD_PROJECT} \
>     --region ${GOOGLE_CLOUD_LOCATION}
> ```

#### Issue: `GOOGLE_CLOUD_LOCATION` not set when running `adk create`

> **Symptom:** `adk create` uses a blank or wrong region value.
>
> **Cause:** The `.env` file wasn't sourced in the current terminal, so `$GOOGLE_CLOUD_LOCATION` is empty.
>
> **Fix:**
> ```bash
> source .env
> echo $GOOGLE_CLOUD_LOCATION  # Should print: global
> uv run adk create jobs_agent \
>     --model gemini-2.5-flash \
>     --project ${GOOGLE_CLOUD_PROJECT} \
>     --region ${GOOGLE_CLOUD_LOCATION}
> ```

---

### Step 4: Seed the Jobs Listing Database

#### Issue: `seed.sql` content not saved correctly

> **Symptom:** Running `psql ... -f seed.sql` returns SQL syntax errors, or the file appears empty when opened.
>
> **Cause:** The `cloudshell edit seed.sql` command opens the file in the Cloud Shell Editor, but you may not have pasted the full SQL content, or the file wasn't saved before running the `psql` command.
>
> **Fix:**
> 1. Open the file: `cloudshell edit seed.sql`
> 2. Verify the file starts with `-- seed.sql` and ends with the last `INSERT` row for GitLab.
> 3. Press **Ctrl+S** to save explicitly.
> 4. Verify the file has content:
>    ```bash
>    wc -l seed.sql  # Should show approximately 60+ lines
>    ```

#### Issue: Cloud SQL instance not ready yet

> **Symptom:** `gcloud sql instances describe jobs-instance --format="value(state)"` returns `PENDING_CREATE` instead of `RUNNABLE`.
>
> **Cause:** The instance creation started in Step 2 hasn't finished yet. Cloud SQL instance creation with `db-custom-1-3840` (dedicated core) takes 5-10 minutes.
>
> **Fix:** Wait and check again periodically:
> ```bash
> watch -n 10 'gcloud sql instances describe jobs-instance --format="value(state)"'
> ```
> Press Ctrl+C once it shows `RUNNABLE`.

#### Issue: IAM policy binding fails

> **Symptom:** The `gcloud projects add-iam-policy-binding` command fails with a permission error.
>
> **Cause:** The service account email wasn't retrieved correctly, or the `$GOOGLE_CLOUD_PROJECT` variable is empty.
>
> **Fix:**
> ```bash
> source .env
> # Verify the service account
> SERVICE_ACCOUNT=$(gcloud sql instances describe jobs-instance --format="value(serviceAccountEmailAddress)")
> echo "Service Account: $SERVICE_ACCOUNT"
>
> # If the above is blank, the instance may not be ready yet
> # If it has a value, retry the binding
> gcloud projects add-iam-policy-binding $GOOGLE_CLOUD_PROJECT \
>   --member="serviceAccount:$SERVICE_ACCOUNT" \
>   --role="roles/aiplatform.user" \
>   --quiet
> ```

#### Issue: Cloud SQL Auth Proxy port 5432 conflict

> **Symptom:** `unable to start: listen tcp 127.0.0.1:5432: bind: address already in use` when starting the proxy.
>
> **Cause:** A previous proxy process or another process is using port 5432.
>
> **Fix:** The codelab mentions this — run:
> ```bash
> lsof -ti:5432 | xargs kill -9 2>/dev/null || fuser -k 5432/tcp 2>/dev/null
> cloud-sql-proxy ${GOOGLE_CLOUD_PROJECT}:${REGION}:jobs-instance --port 5432 &
> ```

#### Issue: `psql` seed script fails with authentication error

> **Symptom:** `psql: error: connection to server at "127.0.0.1", port 5432 failed: FATAL: password authentication failed for user "postgres"`
>
> **Cause:** The `$DB_PASSWORD` variable is empty in the current terminal tab, or it doesn't match the password set during instance creation.
>
> **Fix:**
> ```bash
> source .env
> echo $DB_PASSWORD  # Should print: techjobs-pwd-2025
> ```
> If the variable is set correctly but authentication still fails, the password may not match what was used during instance creation. Reset it:
> ```bash
> gcloud sql users set-password postgres --instance=jobs-instance --password=$DB_PASSWORD
> ```

#### Issue: `psql` command not found

> **Symptom:** `command not found: psql` when trying to run the seed script.
>
> **Cause:** The `psql` client may not be installed in your Cloud Shell instance.
>
> **Fix:**
> ```bash
> sudo apt-get update && sudo apt-get install -y postgresql-client
> ```
> Then retry the `psql` command.

#### Issue: Embedding generation fails silently

> **Symptom:** The background embedding command (`psql ... -c "UPDATE jobs SET description_embedding = embedding(...);" &`) doesn't produce `UPDATE 15` output. When checking later, some or all rows have `f` (false) for `has_embedding`.
>
> **Cause:** The Vertex AI ML integration isn't working. Possible reasons: the IAM binding for `roles/aiplatform.user` hasn't propagated, the `google_ml_integration` extension wasn't installed correctly, or the Vertex AI API isn't enabled.
>
> **Fix:**
> ```bash
> # Verify the extension is installed
> psql "host=127.0.0.1 port=5432 dbname=jobs_db user=postgres password=$DB_PASSWORD" \
>   -c "SELECT extname FROM pg_extension WHERE extname = 'google_ml_integration';"
>
> # If not listed, install it
> psql "host=127.0.0.1 port=5432 dbname=jobs_db user=postgres password=$DB_PASSWORD" \
>   -c "CREATE EXTENSION IF NOT EXISTS google_ml_integration;"
>
> # Retry the embedding generation (foreground this time to see errors)
> psql "host=127.0.0.1 port=5432 dbname=jobs_db user=postgres password=$DB_PASSWORD" \
>   -c "UPDATE jobs SET description_embedding = embedding('gemini-embedding-001', description)::vector;"
> ```
> If it returns an error about Vertex AI access, wait a few minutes for the IAM binding to propagate and retry.

#### Issue: Output opens in a pager and you're stuck

> **Symptom:** After running a `psql` query, the terminal shows results followed by a `:` prompt at the bottom. You can't type commands.
>
> **Cause:** `psql` uses the `less` pager for long output. The `:` is the pager prompt.
>
> **Fix:** Press `q` to exit the pager and return to the terminal. To prevent this in future queries, set:
> ```bash
> export PAGER=cat
> ```

#### Issue: Terminal tabs are getting confusing

> **Symptom:** You have multiple terminal tabs open and don't know which one has the Cloud SQL Auth Proxy or which is free for commands.
>
> **Cause:** The codelab opens a new tab each time a background process is started (Steps 2, 4, 5). By Step 4, you may have 3-4 tabs.
>
> **Fix:** Check what's running in each tab by looking at the last output. The proxy tab shows `Listening on 127.0.0.1:5432`. The embedding tab shows `UPDATE 15` when done. You can also check from any tab:
> ```bash
> # See all background processes
> ss -tlnp | grep -E ':(5432|5000|8000) '
> ```

---

### Step 5: Configure MCP Toolbox for Databases

#### Issue: `tools.yaml` content pasted incorrectly

> **Symptom:** Toolbox fails to start with YAML parsing errors like `yaml: unmarshal errors`, `did not find expected key`, or `mapping values are not allowed in this context`.
>
> **Cause:** The codelab builds `tools.yaml` in four copy-paste steps. Common mistakes: missing `---` separator between documents, pasting a section twice, or indentation errors from copy-paste.
>
> **Fix:**
> 1. Open the file: `cloudshell edit tools.yaml`
> 2. Verify the structure:
>    - First block: `kind: sources`, `name: jobs-db`
>    - `---` separator
>    - Second block: `kind: tools`, `name: search-jobs`
>    - `---` separator
>    - Third block: `kind: tools`, `name: get-job-details`
>    - `---` separator
>    - Fourth block: `kind: embeddingModels`, `name: gemini-embedding`
>    - `---` separator
>    - Fifth block: `kind: tools`, `name: search-jobs-by-description`
>    - `---` separator
>    - Sixth block: `kind: tools`, `name: add-job` (last block, no trailing `---` needed)
> 3. Every `---` must be on its own line, with no leading spaces.
> 4. YAML uses spaces for indentation, not tabs.

#### Issue: Embeddings not ready when verifying

> **Symptom:** The `SELECT title, (description_embedding IS NOT NULL) AS has_embedding` query shows `f` for some rows.
>
> **Cause:** The background embedding generation from Step 4 is still running.
>
> **Fix:** Wait for the process to finish. Check the terminal tab where you ran the embedding command — you should see `UPDATE 15` when it completes. If the terminal was lost, re-run the embedding command:
> ```bash
> psql "host=127.0.0.1 port=5432 dbname=jobs_db user=postgres password=$DB_PASSWORD" \
>   -c "UPDATE jobs SET description_embedding = embedding('gemini-embedding-001', description)::vector WHERE description_embedding IS NULL;"
> ```

#### Issue: Toolbox binary not found or not executable

> **Symptom:** `./toolbox: No such file or directory` or `./toolbox: Permission denied` when trying to start the Toolbox server.
>
> **Cause:** The Toolbox binary download from Step 2 didn't complete, or it downloaded but wasn't made executable.
>
> **Fix:**
> ```bash
> cd ~/build-agent-adk-toolbox-cloudsql
>
> # Check if the file exists and its size
> ls -la toolbox
>
> # If missing or very small (< 1MB), re-download
> rm -f toolbox
> curl -O https://storage.googleapis.com/genai-toolbox/v0.27.0/linux/amd64/toolbox
>
> # Make executable
> chmod +x toolbox
> ```

#### Issue: Toolbox starts but fails to connect to Cloud SQL

> **Symptom:** Toolbox starts but logs show connection errors like `failed to connect to database` or `connection refused`.
>
> **Cause:** Toolbox connects to Cloud SQL using the Cloud SQL connector internally (not the proxy). The required environment variables (`GOOGLE_CLOUD_PROJECT`, `REGION`, `DB_PASSWORD`) may not be exported correctly.
>
> **Fix:**
> ```bash
> # Verify all required variables are set
> echo "Project: $GOOGLE_CLOUD_PROJECT"
> echo "Region: $REGION"
> echo "DB_PASSWORD: $DB_PASSWORD"
> echo "Location: $GOOGLE_CLOUD_LOCATION"
>
> # If any are blank, source the .env
> source .env
>
> # Re-export and restart
> export GOOGLE_CLOUD_PROJECT=$GOOGLE_CLOUD_PROJECT
> export GOOGLE_CLOUD_LOCATION=$GOOGLE_CLOUD_LOCATION
> export GOOGLE_GENAI_USE_VERTEXAI=true
> export DB_PASSWORD=$DB_PASSWORD
> export REGION=$REGION
>
> # Kill old instance
> lsof -ti:5000 | xargs kill -9 2>/dev/null || fuser -k 5000/tcp 2>/dev/null
>
> ./toolbox --tools-file tools.yaml &
> ```

#### Issue: Toolbox initialized 0 tools instead of 4

> **Symptom:** Toolbox starts but logs show `Initialized 0 tools:` instead of `Initialized 4 tools: add-job, search-jobs, get-job-details, search-jobs-by-description`.
>
> **Cause:** The `tools.yaml` file is empty, in the wrong directory, or has YAML syntax errors that silently cause tools to be skipped.
>
> **Fix:**
> ```bash
> # Verify the file exists and has content
> wc -l tools.yaml  # Should be ~80+ lines
>
> # Test YAML syntax
> python3 -c "
> import yaml
> with open('tools.yaml') as f:
>     for doc in yaml.safe_load_all(f):
>         print(doc.get('kind', '?'), doc.get('name', '?'))
> "
> ```
> You should see output listing all 6 resources (1 source, 4 tools, 1 embeddingModel).

#### Issue: `curl` to test Toolbox returns connection refused

> **Symptom:** `curl -s http://localhost:5000/api/toolset` returns `curl: (7) Failed to connect to localhost port 5000: Connection refused`.
>
> **Cause:** The Toolbox server isn't running, crashed, or is still starting up.
>
> **Fix:**
> 1. Check the terminal tab where you started the Toolbox. Look for error messages.
> 2. If the tab was lost, check if it's still running:
>    ```bash
>    ss -tlnp | grep ':5000 '
>    ```
> 3. If not running, restart it (see [Quick Fix #7](#7-toolbox-server-not-running)).

#### Issue: `jq` not found when testing search-jobs

> **Symptom:** `jq: command not found` when running the `curl ... | jq` test command.
>
> **Cause:** `jq` may not be installed in Cloud Shell (though it usually is).
>
> **Fix:** Use `python3` instead:
> ```bash
> curl -s -X POST http://localhost:5000/api/tool/search-jobs/invoke \
>   -H "Content-Type: application/json" \
>   -d '{"role": "Backend", "tech_stack": ""}' | python3 -m json.tool
> ```

---

### Step 6: Build the ADK Agent

#### Issue: `agent.py` not overwritten correctly

> **Symptom:** `SyntaxError`, `ImportError`, or the agent doesn't use Toolbox tools — it responds with generic text instead of querying the database.
>
> **Cause:** The codelab says to "overwrite the content" of `jobs_agent/agent.py`. If you appended instead of replacing, or missed part of the code, the agent won't work correctly.
>
> **Fix:** Open the file and verify it matches the codelab exactly:
> ```bash
> cloudshell edit jobs_agent/agent.py
> ```
> The file should contain:
> 1. Imports: `os`, `LlmAgent`, `ToolboxToolset`
> 2. `TOOLBOX_URL` variable
> 3. `toolbox = ToolboxToolset(TOOLBOX_URL)` instantiation
> 4. `root_agent = LlmAgent(...)` with `tools=[toolbox]`
>
> There should be **only one** `root_agent` definition. Delete any leftover scaffold code from `adk create`.

#### Issue: `ImportError: cannot import name 'ToolboxToolset'`

> **Symptom:** `ImportError: cannot import name 'ToolboxToolset' from 'toolbox_adk'`
>
> **Cause:** The `toolbox-adk` package wasn't installed or the version is wrong.
>
> **Fix:**
> ```bash
> uv pip list | grep toolbox-adk
> # If not listed or wrong version:
> uv add toolbox-adk==0.6.0
> ```

#### Issue: Agent fails to start — Toolbox connection error

> **Symptom:** `uv run adk web` starts but shows errors like `ConnectionRefusedError` or `Failed to connect to Toolbox server` in the terminal.
>
> **Cause:** The Toolbox server is not running on port 5000, or the `TOOLBOX_URL` in `jobs_agent/.env` is wrong.
>
> **Fix:**
> 1. Verify Toolbox is running:
>    ```bash
>    curl -s http://localhost:5000/api/toolset | head -5
>    ```
>    If this fails, restart Toolbox (see [Quick Fix #7](#7-toolbox-server-not-running)).
>
> 2. Verify the agent's `.env` file:
>    ```bash
>    cat jobs_agent/.env
>    ```
>    It should contain `TOOLBOX_URL=http://127.0.0.1:5000`.

#### Issue: Web Preview shows blank page or "Unable to connect"

> **Symptom:** After running `uv run adk web` and opening Web Preview on port 8000, the page is blank or shows a connection error.
>
> **Cause:** The server hasn't fully started, or the wrong port is selected in Web Preview.
>
> **Fix:**
> 1. Check the terminal — wait until you see `Uvicorn running on http://0.0.0.0:8000`.
> 2. In Web Preview, confirm the port is set to **8000** (click the Web Preview button > Preview on port 8000).
> 3. Wait a few seconds and refresh the browser tab.
> 4. Make sure you selected **jobs_agent** from the agent dropdown in the top-left corner.

#### Issue: Agent calls wrong tool or doesn't call any tool

> **Symptom:** You ask "What backend engineering jobs do you have?" but the agent gives a generic response without calling `search-jobs`, or calls `search-jobs-by-description` instead.
>
> **Cause:** The LLM doesn't always call tools deterministically. The agent instruction may not have been copied correctly, or the model interprets the query differently.
>
> **Fix:**
> 1. Try rephrasing: "Search for Backend role jobs" or "Find jobs with role Backend."
> 2. Verify the `instruction` in `root_agent` was copied completely — it should contain guidance on when to use each tool.
> 3. Check the Events panel in the ADK dev UI to see which tool was called and what parameters were passed.

#### Issue: Semantic search returns no results

> **Symptom:** Asking "I want a remote job working on AI" returns no results or an error from `search-jobs-by-description`.
>
> **Cause:** The embeddings weren't generated in Step 4, or the Toolbox embedding model isn't configured correctly.
>
> **Fix:**
> 1. Verify embeddings exist:
>    ```bash
>    psql "host=127.0.0.1 port=5432 dbname=jobs_db user=postgres password=$DB_PASSWORD" \
>      -c "SELECT COUNT(*) FROM jobs WHERE description_embedding IS NOT NULL;"
>    ```
>    Should return `15`. If `0`, run the embedding generation from Step 4.
>
> 2. Check the Toolbox terminal for embedding-related errors. The Toolbox needs `GOOGLE_GENAI_USE_VERTEXAI=true` and `GOOGLE_CLOUD_LOCATION` to call the embedding model.

#### Issue: `add-job` tool fails with casting error

> **Symptom:** When asking the agent to add a job, the terminal shows a SQL error about `CAST($7 AS INTEGER)` or type mismatch.
>
> **Cause:** The LLM passed the `openings` value as a string that can't be cast to an integer (e.g., "two" instead of "2").
>
> **Fix:** Rephrase your request to use numeric values: "Add a new job with 2 openings" instead of "two openings." If the issue persists, check that the `openings` parameter in `tools.yaml` has `type: string` (the CAST in SQL handles the conversion).

#### Issue: Ctrl+C doesn't stop `adk web`

> **Symptom:** Pressing Ctrl+C once doesn't stop the server. The terminal still shows log output.
>
> **Cause:** The codelab says to press Ctrl+C **twice**. The first Ctrl+C initiates graceful shutdown; the second forces termination.
>
> **Fix:** Press Ctrl+C twice quickly. If it still doesn't stop:
> ```bash
> lsof -ti:8000 | xargs kill -9 2>/dev/null || fuser -k 8000/tcp 2>/dev/null
> ```

---

### Step 7: Deploy to Cloud Run

#### Issue: `deploy-toolbox/` directory structure is wrong

> **Symptom:** `gcloud run deploy toolbox-service --source deploy-toolbox/` fails during build with "no such file" errors for `toolbox` or `tools.yaml`.
>
> **Cause:** The `cp toolbox tools.yaml deploy-toolbox/` command wasn't run, or was run from the wrong directory.
>
> **Fix:**
> ```bash
> cd ~/build-agent-adk-toolbox-cloudsql
> ls deploy-toolbox/
> # Should show: Dockerfile  toolbox  tools.yaml
>
> # If files are missing:
> cp toolbox tools.yaml deploy-toolbox/
> ```

#### Issue: Toolbox Cloud Run deployment fails during build

> **Symptom:** `gcloud run deploy toolbox-service` fails with a Cloud Build error, showing messages about Dockerfile syntax or missing files.
>
> **Cause:** The `deploy-toolbox/Dockerfile` content wasn't copied correctly, or the Toolbox binary isn't in the directory.
>
> **Fix:**
> 1. Verify the Dockerfile:
>    ```bash
>    cat deploy-toolbox/Dockerfile
>    ```
>    Should start with `FROM debian:bookworm-slim` and end with the `CMD` line.
>
> 2. Verify the toolbox binary exists and isn't empty:
>    ```bash
>    ls -lh deploy-toolbox/toolbox  # Should be ~30-50MB
>    ```

#### Issue: Toolbox service URL returns empty

> **Symptom:** After deploying the Toolbox, `TOOLBOX_URL` is empty when running `gcloud run services describe`.
>
> **Cause:** The deployment is still in progress, or it failed.
>
> **Fix:**
> ```bash
> # Check deployment status
> gcloud run services list --region=$REGION
>
> # If toolbox-service is listed, get its URL
> gcloud run services describe toolbox-service \
>   --region=$REGION \
>   --format='value(status.url)'
>
> # If not listed, the deploy command may still be running in another tab
> # Check that tab for completion or errors
> ```

#### Issue: Deployed Toolbox verification fails

> **Symptom:** `curl -s "$TOOLBOX_URL/api/toolset"` returns an error or empty response.
>
> **Cause:** The Toolbox service failed to start on Cloud Run. Common reasons: environment variables not set correctly, or the Cloud SQL connection fails from Cloud Run.
>
> **Fix:**
> ```bash
> # Check the service logs
> gcloud run services logs read toolbox-service --region=$REGION --limit=20
>
> # Common issue: environment variables missing
> # Redeploy with explicit variables
> gcloud run deploy toolbox-service \
>   --source deploy-toolbox/ \
>   --region $REGION \
>   --set-env-vars "DB_PASSWORD=$DB_PASSWORD,GOOGLE_CLOUD_PROJECT=$GOOGLE_CLOUD_PROJECT,REGION=$REGION,GOOGLE_CLOUD_LOCATION=$GOOGLE_CLOUD_LOCATION,GOOGLE_GENAI_USE_VERTEXAI=true" \
>   --allow-unauthenticated \
>   --quiet
> ```

#### Issue: Agent deployment fails with Dockerfile errors

> **Symptom:** `gcloud run deploy jobs-agent --source .` fails during Cloud Build. Errors mention missing `pyproject.toml`, `uv.lock`, or build failures.
>
> **Cause:** The Dockerfile or `.dockerignore` wasn't created correctly, or `uv.lock` doesn't exist.
>
> **Fix:**
> ```bash
> cd ~/build-agent-adk-toolbox-cloudsql
>
> # Verify uv.lock exists
> ls uv.lock
> # If missing, generate it:
> uv lock
>
> # Verify Dockerfile exists and has correct content
> cat Dockerfile
> # Should start with: FROM ghcr.io/astral-sh/uv:python3.12-trixie-slim
>
> # Verify .dockerignore exists
> cat .dockerignore
> # Should list .venv/, __pycache__/, *.pyc, .env, etc.
> ```

#### Issue: Agent Cloud Run service returns errors when testing

> **Symptom:** The deployed agent UI loads but queries fail with errors. The agent can't reach the Toolbox service.
>
> **Cause:** The `TOOLBOX_URL` environment variable passed to the agent service is wrong or empty.
>
> **Fix:**
> ```bash
> # Verify what TOOLBOX_URL is set to
> gcloud run services describe jobs-agent \
>   --region=$REGION \
>   --format='value(spec.template.spec.containers[0].env)'
>
> # If TOOLBOX_URL is wrong, update the deployment
> TOOLBOX_URL=$(gcloud run services describe toolbox-service \
>   --region=$REGION \
>   --format='value(status.url)')
>
> gcloud run deploy jobs-agent \
>   --source . \
>   --region $REGION \
>   --set-env-vars "TOOLBOX_URL=$TOOLBOX_URL,GOOGLE_CLOUD_PROJECT=$GOOGLE_CLOUD_PROJECT,GOOGLE_CLOUD_LOCATION=$GOOGLE_CLOUD_LOCATION,GOOGLE_GENAI_USE_VERTEXAI=TRUE" \
>   --allow-unauthenticated \
>   --quiet
> ```

#### Issue: Deployment takes too long

> **Symptom:** `gcloud run deploy` is running for more than 10 minutes without completing.
>
> **Cause:** Cloud Build is building the Docker image, which involves downloading base images and installing dependencies. First deployments take longer because Artifact Registry and Cloud Build infrastructure must be provisioned.
>
> **Fix:** This is normal for the first deployment. The build includes:
> - Creating an Artifact Registry repository (first time only)
> - Pulling the base image
> - Installing Python dependencies with `uv sync`
>
> Check progress in another tab:
> ```bash
> gcloud builds list --region=$REGION --limit=1
> ```
> If the build failed, check logs:
> ```bash
> gcloud builds log $(gcloud builds list --region=$REGION --limit=1 --format='value(id)')
> ```

---

### Step 8: Congratulations / Clean Up

#### Issue: `gcloud projects delete` asks for confirmation

> **Symptom:** The command prompts "Do you want to continue?" and waits for input.
>
> **Cause:** The `--quiet` flag wasn't used in the codelab command.
>
> **Fix:** Type `Y` and press Enter to confirm.

#### Issue: Individual resource deletion fails

> **Symptom:** `gcloud sql instances delete jobs-instance` fails or takes a very long time.
>
> **Cause:** The instance may have active connections or ongoing operations.
>
> **Fix:**
> ```bash
> # Kill any local proxy processes first
> lsof -ti:5432 | xargs kill -9 2>/dev/null || fuser -k 5432/tcp 2>/dev/null
>
> # Then delete
> gcloud sql instances delete jobs-instance --quiet
> ```
> Cloud SQL instance deletion can take several minutes. Wait for the command to complete.

#### Issue: Artifact Registry deletion fails

> **Symptom:** `gcloud artifacts repositories delete cloud-run-source-deploy` fails with `NOT_FOUND`.
>
> **Cause:** The repository name or location doesn't match. Cloud Build may have used a different repository name.
>
> **Fix:**
> ```bash
> # List all artifact repositories
> gcloud artifacts repositories list --location=$REGION
>
> # Delete using the actual name shown
> gcloud artifacts repositories delete <actual-name> --location=$REGION --quiet
> ```

---

## General FAQ

### Q: Which Google account should I use — personal Gmail or Workspace?

**A:** For trial credits, use your **personal Gmail** account. Workspace accounts may have organizational policies that restrict project creation or billing. If you're in an instructor-led workshop, follow the instructor's guidance.

### Q: What's the difference between `GOOGLE_CLOUD_LOCATION` and `REGION`?

**A:** `GOOGLE_CLOUD_LOCATION` (set to `global`) is used only for Vertex AI / Gemini API calls — both in the ADK agent and in the Toolbox embedding model. `REGION` (set to `us-central1`) is used for all other GCP products like Cloud SQL, Cloud Run, and Artifact Registry. These are separate because Vertex AI and infrastructure resources have different regional availability.

### Q: Why does the codelab open so many terminal tabs?

**A:** The codelab runs several background processes that produce continuous output: Cloud SQL instance creation, Toolbox binary download, Cloud SQL Auth Proxy, embedding generation, and the Toolbox server. Each time a background process is started, a new tab is opened so its output doesn't interfere with your commands. By Step 6, you may have 4-5 tabs open. Only the most recent tab needs environment variables — the background processes in other tabs continue running independently.

### Q: Can I use a different Gemini model instead of `gemini-2.5-flash`?

**A:** Yes, but you'd need to modify both the `adk create` command (Step 3) and the `model` parameter in the `root_agent` definition in `agent.py`. Use the same model name in both places. Note that model availability varies by region.

### Q: What's the difference between the Cloud SQL Auth Proxy and Toolbox's Cloud SQL connection?

**A:** The Cloud SQL Auth Proxy (`cloud-sql-proxy`) is used for **your direct access** to the database from Cloud Shell — for seeding data, running queries, and generating embeddings via `psql`. Toolbox handles its **own connection** to Cloud SQL internally using the Cloud SQL connector library — it doesn't use the proxy. Both methods authenticate via Application Default Credentials, but they're separate processes.

### Q: Can I use a different database password?

**A:** Yes. Change the `DB_PASSWORD` value in your `.env` file **before** running the Cloud SQL instance creation command. If you already created the instance with `techjobs-pwd-2025`, reset the password:
```bash
gcloud sql users set-password postgres --instance=jobs-instance --password=YOUR_NEW_PASSWORD
echo "DB_PASSWORD=YOUR_NEW_PASSWORD" >> .env
source .env
```

### Q: Why does the codelab use `uv run adk web` instead of just `adk web`?

**A:** `uv run` executes the command within the project's virtual environment managed by `uv`. This ensures the correct Python version and dependencies are used without manually activating a virtual environment.

### Q: Can I modify `tools.yaml` without redeploying the agent?

**A:** Yes, that's one of the key benefits of Toolbox. When running locally, edit `tools.yaml` and restart the Toolbox server — the agent code doesn't change. When deployed, you only need to redeploy the `toolbox-service` Cloud Run service, not the `jobs-agent` service. The agent dynamically loads tools from the Toolbox server at startup.

### Q: Do I need to keep the Cloud Shell tab open the entire time?

**A:** Yes, while background processes are running (Cloud SQL Auth Proxy, Toolbox server, `adk web`), the Cloud Shell terminal must remain open. If the session times out, all background processes stop. The Cloud SQL instance and deployed Cloud Run services continue running independently. See [Environment Recovery](#environment-recovery) to resume.

---

## Environment Recovery

Use this section if you've been disconnected from Cloud Shell, your session timed out, or you need to get back to a known-good state.

### Recovering from Cloud Shell Disconnection

If your Cloud Shell session was disconnected or timed out:

1. **Re-open Cloud Shell** at [https://ide.cloud.google.com](https://ide.cloud.google.com). Click **View > Terminal** to open the terminal.

2. **Navigate to your working directory:**
   ```bash
   cd ~/build-agent-adk-toolbox-cloudsql
   ```

3. **Reload your environment variables and project configuration:**
   ```bash
   bash setup_verify_trial_project.sh && source .env
   ```
   Verify the yellow project ID text reappears in the terminal prompt.

4. **Verify key environment variables:**
   ```bash
   echo "Project: $GOOGLE_CLOUD_PROJECT"
   echo "Region: $REGION"
   echo "Location: $GOOGLE_CLOUD_LOCATION"
   echo "DB Password: $DB_PASSWORD"
   ```
   All four should print non-empty values.

5. **Restart the Cloud SQL Auth Proxy** (if you were on Step 4 or later and need direct database access):
   ```bash
   # Kill any stale proxy processes
   lsof -ti:5432 | xargs kill -9 2>/dev/null || fuser -k 5432/tcp 2>/dev/null

   # Restart the proxy
   cloud-sql-proxy ${GOOGLE_CLOUD_PROJECT}:${REGION}:jobs-instance --port 5432 &
   ```

6. **Restart the Toolbox server** (if you were on Step 5 or later):
   ```bash
   # Kill any stale processes
   lsof -ti:5000 | xargs kill -9 2>/dev/null || fuser -k 5000/tcp 2>/dev/null

   # Export required variables
   export GOOGLE_CLOUD_PROJECT=$GOOGLE_CLOUD_PROJECT
   export GOOGLE_CLOUD_LOCATION=$GOOGLE_CLOUD_LOCATION
   export GOOGLE_GENAI_USE_VERTEXAI=true
   export DB_PASSWORD=$DB_PASSWORD
   export REGION=$REGION

   # Start Toolbox
   ./toolbox --tools-file tools.yaml &
   ```
   Wait for `Server ready to serve!` before proceeding.

7. **Kill stale processes on port 8000** (if you were running `adk web`):
   ```bash
   lsof -ti:8000 | xargs kill -9 2>/dev/null || fuser -k 8000/tcp 2>/dev/null
   ```

8. **Resume where you left off** — your files, Cloud SQL instance, and database all persist across Cloud Shell reconnections. Only background processes (proxy, Toolbox, `adk web`) need to be restarted.

### Verifying Your Environment State

Run these checks to confirm your environment is properly configured:

```bash
# Check project
echo "Project: $GOOGLE_CLOUD_PROJECT"

# Check region variables
echo "Region: $REGION"
echo "Location: $GOOGLE_CLOUD_LOCATION"

# Check database password
echo "DB Password: $DB_PASSWORD"

# Verify gcloud project
gcloud config get-value project

# Check if Cloud SQL instance exists and is running
gcloud sql instances describe jobs-instance --format="value(state)" 2>/dev/null || echo "Instance NOT FOUND"

# Check if Cloud SQL Auth Proxy is running
ss -tlnp | grep ':5432 ' && echo "Proxy: RUNNING" || echo "Proxy: NOT RUNNING"

# Check if Toolbox is running
ss -tlnp | grep ':5000 ' && echo "Toolbox: RUNNING" || echo "Toolbox: NOT RUNNING"

# Check if ADK web is running
ss -tlnp | grep ':8000 ' && echo "ADK Web: RUNNING" || echo "ADK Web: NOT RUNNING"

# Verify key files exist
ls toolbox tools.yaml jobs_agent/agent.py 2>/dev/null && echo "Key files: EXIST" || echo "Key files: SOME MISSING"
```

### Starting Over from a Specific Step

If you need to restart from a particular step:

- **From Step 1 (Set Up Your Environment):**
  ```bash
  cd ~
  rm -rf build-agent-adk-toolbox-cloudsql
  mkdir -p ~/build-agent-adk-toolbox-cloudsql
  cloudshell workspace ~/build-agent-adk-toolbox-cloudsql && cd ~/build-agent-adk-toolbox-cloudsql
  ```
  Then follow Step 1 from the beginning.

- **From Step 2 (Create the Database Instance):**
  ```bash
  cd ~/build-agent-adk-toolbox-cloudsql
  source .env
  # Check if instance already exists
  gcloud sql instances describe jobs-instance --format="value(state)" 2>/dev/null
  # If it exists and is RUNNABLE, skip to Step 3
  # If not, re-run the creation command from the codelab
  ```

- **From Step 3 (Initialize the Agent Project):**
  ```bash
  cd ~/build-agent-adk-toolbox-cloudsql
  source .env
  rm -rf jobs_agent .venv pyproject.toml uv.lock
  uv init
  uv add google-adk==1.25.0 toolbox-adk==0.6.0
  uv run adk create jobs_agent \
      --model gemini-2.5-flash \
      --project ${GOOGLE_CLOUD_PROJECT} \
      --region ${GOOGLE_CLOUD_LOCATION}
  ```
  Then continue with Step 3's remaining instructions.

- **From Step 4 (Seed the Jobs Listing Database):**
  ```bash
  cd ~/build-agent-adk-toolbox-cloudsql
  bash setup_verify_trial_project.sh && source .env
  # Verify Cloud SQL instance is ready
  gcloud sql instances describe jobs-instance --format="value(state)"
  # If RUNNABLE, start the proxy and run seeding
  lsof -ti:5432 | xargs kill -9 2>/dev/null || fuser -k 5432/tcp 2>/dev/null
  cloud-sql-proxy ${GOOGLE_CLOUD_PROJECT}:${REGION}:jobs-instance --port 5432 &
  # Wait for proxy, then check if data already exists
  psql "host=127.0.0.1 port=5432 dbname=jobs_db user=postgres password=$DB_PASSWORD" \
    -c "SELECT COUNT(*) FROM jobs;" 2>/dev/null
  # If 15 rows exist, skip seeding. If 0 or error, re-run seed.sql
  ```

- **From Step 5 (Configure MCP Toolbox for Databases):**
  ```bash
  cd ~/build-agent-adk-toolbox-cloudsql
  bash setup_verify_trial_project.sh && source .env
  # Ensure toolbox binary exists
  ls -la toolbox
  # If missing, re-download
  # Ensure tools.yaml exists
  ls -la tools.yaml
  # Kill stale processes
  lsof -ti:5000 | xargs kill -9 2>/dev/null || fuser -k 5000/tcp 2>/dev/null
  # Start Toolbox
  export GOOGLE_CLOUD_PROJECT=$GOOGLE_CLOUD_PROJECT
  export GOOGLE_CLOUD_LOCATION=$GOOGLE_CLOUD_LOCATION
  export GOOGLE_GENAI_USE_VERTEXAI=true
  export DB_PASSWORD=$DB_PASSWORD
  export REGION=$REGION
  ./toolbox --tools-file tools.yaml &
  ```

- **From Step 6 (Build the ADK Agent):**
  ```bash
  cd ~/build-agent-adk-toolbox-cloudsql
  bash setup_verify_trial_project.sh && source .env
  # Ensure Toolbox is running
  curl -s http://localhost:5000/api/toolset | head -5
  # If not running, restart it (see Step 5 recovery above)
  # Kill stale ADK processes
  lsof -ti:8000 | xargs kill -9 2>/dev/null || fuser -k 8000/tcp 2>/dev/null
  # Start the agent
  uv run adk web
  ```

- **From Step 7 (Deploy to Cloud Run):**
  ```bash
  cd ~/build-agent-adk-toolbox-cloudsql
  bash setup_verify_trial_project.sh && source .env
  # Check if services are already deployed
  gcloud run services list --region=$REGION
  # If already deployed, get URLs:
  TOOLBOX_URL=$(gcloud run services describe toolbox-service --region=$REGION --format='value(status.url)')
  AGENT_URL=$(gcloud run services describe jobs-agent --region=$REGION --format='value(status.url)')
  echo "Toolbox: $TOOLBOX_URL"
  echo "Agent: $AGENT_URL"
  ```

- **Complete restart (nuclear option):**
  ```bash
  cd ~
  rm -rf build-agent-adk-toolbox-cloudsql
  # Delete Cloud Run services
  gcloud run services delete jobs-agent --region=us-central1 --quiet 2>/dev/null
  gcloud run services delete toolbox-service --region=us-central1 --quiet 2>/dev/null
  # Delete Cloud SQL instance
  gcloud sql instances delete jobs-instance --quiet 2>/dev/null
  # Delete Artifact Registry
  gcloud artifacts repositories delete cloud-run-source-deploy --location=us-central1 --quiet 2>/dev/null
  ```
  Then start from Step 1 of the codelab.
