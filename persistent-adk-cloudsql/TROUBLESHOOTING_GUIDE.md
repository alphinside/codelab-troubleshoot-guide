# Troubleshooting Guide: Building Persistent AI Agents with ADK and CloudSQL

> **Codelab URL:** https://codelabs.developers.google.com/persistent-adk-cloudsql
>
> **Last updated:** 2026-02-24

This guide helps you diagnose and fix common issues encountered while following the **Building Persistent AI Agents with ADK and CloudSQL** codelab. If you're stuck, start with the [Quick Fixes](#quick-fixes) section — it covers the most frequent problems.

---

## Table of Contents

- [Quick Fixes](#quick-fixes)
- [Step-by-Step Troubleshooting](#step-by-step-troubleshooting)
  - [Step 1: Set Up Your Environment](#step-1-set-up-your-environment)
  - [Step 2: Set Up Cloud SQL](#step-2-set-up-cloud-sql)
  - [Step 3: Build the Cafe Concierge Agent](#step-3-build-the-cafe-concierge-agent)
  - [Step 4: Add Stateful Order Management](#step-4-add-stateful-order-management)
  - [Step 5: Test the Agent with the ADK Dev UI](#step-5-test-the-agent-with-the-adk-dev-ui)
  - [Step 6: Observe Local Storage Limitation](#step-6-observe-local-storage-limitation)
  - [Step 7: Revisit Database Setup](#step-7-revisit-database-setup)
  - [Step 8: Verify Persistent Memory Across Sessions](#step-8-verify-persistent-memory-across-sessions)
  - [Step 9: Congratulations / Clean Up](#step-9-congratulations--clean-up)
- [General FAQ](#general-faq)
- [Environment Recovery](#environment-recovery)

---

## Quick Fixes

The most common issues across all steps. **Check these first** before diving into step-specific troubleshooting.

### 1. Environment variables are empty after switching terminal tabs

> **Symptom:** Commands like `echo $GOOGLE_CLOUD_PROJECT` or `echo $DB_PASSWORD` return blank. `gcloud` commands fail with project-related errors.
>
> **Cause:** Each terminal tab in Cloud Shell has its own shell session. Environment variables from `.env` are only loaded in the tab where you ran `source .env`. Opening a new tab (as instructed in Step 2) starts a clean session.
>
> **Fix:**
> ```bash
> cd ~/build-agent-adk-cloudsql
> source .env
> ```
> Verify with `echo $GOOGLE_CLOUD_PROJECT` — it should print your project ID.

### 2. Port 8000 already in use

> **Symptom:** `ERROR: [Errno 98] Address already in use` or `ERROR: Could not bind to port 8000` when running `uv run adk web`.
>
> **Cause:** A previous `adk web` process wasn't stopped with Ctrl+C, or is still running in another terminal tab.
>
> **Fix:**
> ```bash
> lsof -ti:8000 | xargs kill -9 2>/dev/null || fuser -k 8000/tcp 2>/dev/null
> ```
> If both commands fail, try: `uv run adk web --port 8001` and use port 8001 in Web Preview instead.

### 3. Cloud Shell session disconnected

> **Symptom:** Cloud Shell terminal is unresponsive, shows "Reconnecting...", or you return to a fresh terminal without your environment variables.
>
> **Cause:** Cloud Shell timed out due to inactivity (default: 20 minutes idle) or a network interruption.
>
> **Fix:** See the [Environment Recovery](#environment-recovery) section for full recovery steps. The short version:
> ```bash
> cd ~/build-agent-adk-cloudsql
> bash setup_verify_trial_project.sh && source .env
> ```

### 4. Cloud SQL Auth Proxy not running

> **Symptom:** Connecting to the database fails with `could not connect to server: Connection refused` or `Is the server running on host "127.0.0.1" and accepting TCP/IP connections on port 5432?`
>
> **Cause:** The Cloud SQL Auth Proxy background process was killed — either by Cloud Shell disconnection, closing the terminal tab, or pressing Ctrl+C in the wrong tab.
>
> **Fix:**
> ```bash
> # Check if the proxy is running
> ss -tlnp | grep ':5432 '
>
> # If not running, restart it
> cd ~/build-agent-adk-cloudsql
> source .env
> cloud-sql-proxy ${GOOGLE_CLOUD_PROJECT}:${REGION}:cafe-concierge-db --port 5432 &
> ```

### 5. Port 5432 already in use (Cloud SQL Auth Proxy conflict)

> **Symptom:** `unable to start: listen tcp 127.0.0.1:5432: bind: address already in use` when starting the Cloud SQL Auth Proxy.
>
> **Cause:** A previous proxy process is still running, or another process (like a local PostgreSQL server) is using port 5432.
>
> **Fix:**
> ```bash
> lsof -ti:5432 | xargs kill -9 2>/dev/null || fuser -k 5432/tcp 2>/dev/null
> cloud-sql-proxy ${GOOGLE_CLOUD_PROJECT}:${REGION}:cafe-concierge-db --port 5432 &
> ```

### 6. Proxy restart command has wrong syntax (Step 8 bug)

> **Symptom:** Running the proxy check/restart snippet from Step 8 results in `ERROR: invalid instance connection name` because the connection string looks like `your-project:cafe-concierge-db` (missing region).
>
> **Cause:** The codelab's proxy restart snippet in Step 8 contains a typo: `{$REGION}` instead of `${REGION}`. The curly braces are in the wrong position, causing the variable to not expand correctly.
>
> **Fix:** Use this corrected command instead:
> ```bash
> cloud-sql-proxy ${GOOGLE_CLOUD_PROJECT}:${REGION}:cafe-concierge-db --port 5432 &
> ```

### 7. Vertex AI API errors when running the agent

> **Symptom:** Agent responds with `Vertex AI API has not been used in project` or `403 Permission Denied`.
>
> **Cause:** The Vertex AI API wasn't enabled in Step 1, or the enablement hasn't propagated yet.
>
> **Fix:**
> ```bash
> gcloud services enable aiplatform.googleapis.com
> ```
> Wait for `Operation/... finished successfully` before retrying.

### 8. Agent code errors after editing `agent.py`

> **Symptom:** `SyntaxError`, `IndentationError`, or `NameError` when running `uv run adk web`.
>
> **Cause:** Copy-paste errors when adding code in Steps 3-4. Common issues: missing imports, wrong indentation, functions placed after `root_agent` instead of before it.
>
> **Fix:** Open the file and compare against the expected structure:
> ```bash
> cloudshell edit cafe_concierge/agent.py
> ```
> The file should contain, in order: imports, `CAFE_MENU` dict, five tool functions (`get_menu`, `place_order`, `get_order_summary`, `set_dietary_preference`, `get_dietary_preferences`), then the `root_agent` definition. Check the terminal output for the specific error line number.

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

> **Symptom:** After running `cloudshell workspace ~/build-agent-adk-cloudsql && cd ~/build-agent-adk-cloudsql`, the Cloud Shell Editor file tree still shows a different directory.
>
> **Cause:** The `cloudshell workspace` command sometimes requires a page refresh to take effect in the editor.
>
> **Fix:**
> 1. Refresh the Cloud Shell browser tab.
> 2. If the editor still shows the wrong directory, click **File > Open Folder** in the Cloud Shell Editor and navigate to `~/build-agent-adk-cloudsql`.
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
> gcloud services enable aiplatform.googleapis.com sqladmin.googleapis.com compute.googleapis.com
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

### Step 2: Set Up Cloud SQL

#### Issue: Cloud SQL instance creation fails immediately

> **Symptom:** The `gcloud sql instances create` command returns an error about quotas, permissions, or the Compute Engine API not being enabled.
>
> **Cause:** The Compute Engine API or Cloud SQL Admin API wasn't enabled in Step 1, or your project has hit a resource quota.
>
> **Fix:**
> ```bash
> # Ensure all required APIs are enabled
> gcloud services enable sqladmin.googleapis.com compute.googleapis.com
>
> # Retry instance creation
> gcloud sql instances create cafe-concierge-db \
>   --database-version=POSTGRES_17 \
>   --edition=ENTERPRISE \
>   --region=${REGION} \
>   --availability-type=ZONAL \
>   --project=${GOOGLE_CLOUD_PROJECT} \
>   --tier=db-f1-micro \
>   --root-password=${DB_PASSWORD} \
>   --quiet &
> ```

#### Issue: Background process output floods the terminal

> **Symptom:** While typing in the terminal, Cloud SQL creation progress messages appear and disrupt your input.
>
> **Cause:** The `&` suffix runs the command in the background, but its stdout/stderr still prints to the current terminal.
>
> **Fix:** This is expected behavior. Open a **new terminal tab** (click the `+` icon in Cloud Shell) as instructed. Don't forget to re-source your environment in the new tab:
> ```bash
> cd ~/build-agent-adk-cloudsql
> bash setup_verify_trial_project.sh && source .env
> ```

#### Issue: `DB_PASSWORD` variable is empty in the new terminal tab

> **Symptom:** In the new terminal tab, `echo $DB_PASSWORD` returns blank.
>
> **Cause:** Environment variables are per-tab. The `DB_PASSWORD` was written to `.env` in the first tab, but the new tab hasn't sourced that file.
>
> **Fix:**
> ```bash
> cd ~/build-agent-adk-cloudsql
> source .env
> echo $DB_PASSWORD  # Should print: cafe-agent-pwd-2025
> ```

#### Issue: Instance already exists

> **Symptom:** `ERROR: (gcloud.sql.instances.create) Resource already exists` when running the create command.
>
> **Cause:** You ran the creation command twice, or a previous attempt partially completed.
>
> **Fix:** Check the instance status:
> ```bash
> gcloud sql instances describe cafe-concierge-db --format="value(state)"
> ```
> If it returns `RUNNABLE`, the instance is ready — skip to the next step. If it returns `PENDING_CREATE`, wait for it to finish. If it shows an error state:
> ```bash
> gcloud sql instances delete cafe-concierge-db --quiet
> # Then re-run the creation command
> ```

---

### Step 3: Build the Cafe Concierge Agent

#### Issue: `uv init` fails or creates unexpected files

> **Symptom:** `uv init` fails with an error or doesn't create `pyproject.toml`.
>
> **Cause:** `uv init` may fail if there's already a `pyproject.toml` in the directory from a previous attempt.
>
> **Fix:**
> ```bash
> # If retrying, remove old project files first
> rm -f pyproject.toml uv.lock
> rm -rf .venv
> uv init
> ```

#### Issue: `uv add google-adk==1.25.0 asyncpg` fails

> **Symptom:** Dependency resolution errors or network timeouts during `uv add`.
>
> **Cause:** Network issues in Cloud Shell, or a version conflict with the Python version.
>
> **Fix:**
> ```bash
> # Retry with increased timeout
> uv add google-adk==1.25.0 asyncpg --timeout 120
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

#### Issue: `GOOGLE_CLOUD_LOCATION` not set when running `adk create`

> **Symptom:** `adk create` uses a blank or wrong region value.
>
> **Cause:** The `.env` file wasn't sourced in the current terminal, so `$GOOGLE_CLOUD_LOCATION` is empty.
>
> **Fix:**
> ```bash
> source .env
> echo $GOOGLE_CLOUD_LOCATION  # Should print: global
> uv run adk create cafe_concierge \
>     --model gemini-2.5-flash \
>     --project ${GOOGLE_CLOUD_PROJECT} \
>     --region ${GOOGLE_CLOUD_LOCATION}
> ```

#### Issue: `cafe_concierge` directory already exists

> **Symptom:** `adk create cafe_concierge` fails because the directory already exists from a previous attempt.
>
> **Cause:** You ran `adk create` before (possibly with an error) and the directory was partially created.
>
> **Fix:**
> ```bash
> rm -rf cafe_concierge
> uv run adk create cafe_concierge \
>     --model gemini-2.5-flash \
>     --project ${GOOGLE_CLOUD_PROJECT} \
>     --region ${GOOGLE_CLOUD_LOCATION}
> ```

#### Issue: Web Preview shows blank page or "Unable to connect"

> **Symptom:** After running `uv run adk web` and opening Web Preview on port 8000, the page is blank or shows a connection error.
>
> **Cause:** The server hasn't fully started, or the wrong port is selected in Web Preview.
>
> **Fix:**
> 1. Check the terminal — wait until you see `Uvicorn running on http://0.0.0.0:8000`.
> 2. In Web Preview, confirm the port is set to **8000**.
> 3. Wait a few seconds and refresh the browser tab.
> 4. Make sure you selected `cafe_concierge` from the agent dropdown in the top-left corner of the dev UI.

#### Issue: Agent doesn't respond or returns errors

> **Symptom:** After selecting `cafe_concierge` and typing a message, the chat returns an error or hangs.
>
> **Cause:** Vertex AI API issues, wrong project ID in the agent's `.env`, or model access issues.
>
> **Fix:**
> ```bash
> # Check the agent's .env file
> cat cafe_concierge/.env
> ```
> Verify `GOOGLE_CLOUD_PROJECT` and `GOOGLE_CLOUD_LOCATION` are set correctly. If blank or wrong:
> ```bash
> # The adk create command should have populated this, but verify:
> echo "GOOGLE_CLOUD_PROJECT=${GOOGLE_CLOUD_PROJECT}" > cafe_concierge/.env
> echo "GOOGLE_CLOUD_LOCATION=${GOOGLE_CLOUD_LOCATION}" >> cafe_concierge/.env
> echo "GOOGLE_GENAI_USE_VERTEXAI=1" >> cafe_concierge/.env
> ```

---

### Step 4: Add Stateful Order Management

#### Issue: Functions placed in the wrong location in `agent.py`

> **Symptom:** `NameError: name 'place_order' is not defined` or similar errors when running the agent.
>
> **Cause:** The four new tool functions were added **after** the `root_agent` definition instead of **above** it. Python reads top-to-bottom — functions must be defined before they're referenced.
>
> **Fix:** Open `cafe_concierge/agent.py` and ensure this order:
> 1. Imports (`from google.adk.agents import LlmAgent` and `from google.adk.tools import ToolContext`)
> 2. `CAFE_MENU` dictionary
> 3. `get_menu()` function
> 4. `place_order()` function
> 5. `get_order_summary()` function
> 6. `set_dietary_preference()` function
> 7. `get_dietary_preferences()` function
> 8. `root_agent = LlmAgent(...)` — **last** in the file

#### Issue: Old `root_agent` definition not replaced

> **Symptom:** The agent runs but doesn't use the new tools (can't place orders or save preferences). Or you get a `SyntaxError` because there are two `root_agent` definitions.
>
> **Cause:** The codelab says to "replace the existing `root_agent` definition" but you added the new one without removing the old one, or didn't replace it at all.
>
> **Fix:** Open `cafe_concierge/agent.py` and search for `root_agent`. There should be exactly **one** `root_agent = LlmAgent(...)` block, and it should list all five tools: `get_menu`, `place_order`, `get_order_summary`, `set_dietary_preference`, `get_dietary_preferences`.

#### Issue: `IndentationError` in Python code

> **Symptom:** `IndentationError: unexpected indent` or `IndentationError: expected an indented block` when running the agent.
>
> **Cause:** Copy-paste from the codelab sometimes introduces mixed tabs/spaces or incorrect indentation levels — especially for the function bodies.
>
> **Fix:** Python requires consistent indentation (4 spaces per level). Open the file in the Cloud Shell Editor and check that:
> - Function bodies are indented 4 spaces from `def`
> - Code inside `if`/`for` blocks is indented 4 more spaces
> - No tabs are mixed with spaces
>
> The Cloud Shell Editor shows whitespace issues — look for red squiggly underlines.

#### Issue: `ImportError: cannot import name 'ToolContext'`

> **Symptom:** `ImportError: cannot import name 'ToolContext' from 'google.adk.tools'`
>
> **Cause:** The `ToolContext` import was missing or the `google-adk` package version doesn't include it.
>
> **Fix:** Ensure the import line at the top of `agent.py` reads:
> ```python
> from google.adk.tools import ToolContext
> ```
> If the import still fails, verify the ADK version:
> ```bash
> uv pip show google-adk
> ```
> It should show version `1.25.0`. If not:
> ```bash
> uv add google-adk==1.25.0
> ```

---

### Step 5: Test the Agent with the ADK Dev UI

#### Issue: Agent doesn't call tools (responds with generic text instead)

> **Symptom:** You ask "What's on the menu?" but the agent gives a generic response instead of calling `get_menu`. Or you say "I'm lactose intolerant" but the agent doesn't call `set_dietary_preference`.
>
> **Cause:** The LLM doesn't always call tools deterministically. The agent instruction or tool docstrings may not have been copied correctly, or the model is choosing to respond conversationally.
>
> **Fix:**
> 1. Try rephrasing: "Show me the full menu using the menu tool" or "Please save my dietary preference: lactose intolerant."
> 2. Verify the `instruction` in `root_agent` includes the line about "ALWAYS save it using the set_dietary_preference tool."
> 3. Verify all five tools are listed in the `tools=[...]` array.

#### Issue: Events panel doesn't show `state_delta`

> **Symptom:** You click on a `tool_response` event but don't see a `state_delta` field.
>
> **Cause:** The tool didn't write to `tool_context.state`, which means the function code may be incorrect — the state write line is missing.
>
> **Fix:** Check `cafe_concierge/agent.py` and verify that:
> - `place_order` contains `tool_context.state["current_order"] = order`
> - `set_dietary_preference` contains `tool_context.state["user:dietary_preferences"] = existing`
>
> After fixing, restart the agent (Ctrl+C, then `uv run adk web`).

#### Issue: "New Session" button not visible

> **Symptom:** You can't find a way to start a new session in the ADK dev UI.
>
> **Cause:** The button location varies by ADK version. It may be labeled differently or located in an unexpected position.
>
> **Fix:** Look for a `+` icon or "New Session" button near the session ID at the top of the chat interface. If you can't find it, refresh the page — this creates a new session automatically.

#### Issue: Cross-session state doesn't persist (Conversation 2 doesn't remember preferences)

> **Symptom:** After starting a new session, the agent doesn't remember your dietary preferences from Conversation 1.
>
> **Cause:** The `user:` prefix is missing from the state key in `set_dietary_preference`, or the user ID changed between sessions.
>
> **Fix:** Verify that `set_dietary_preference` writes to `tool_context.state["user:dietary_preferences"]` (with the `user:` prefix). Also check the State tab — if you see `dietary_preferences` without the `user:` prefix, the key is session-scoped and won't carry over.

---

### Step 6: Observe Local Storage Limitation

#### Issue: Agent still remembers preferences after deleting `session.db`

> **Symptom:** After deleting `session.db` and restarting, the agent somehow still knows your preferences.
>
> **Cause:** You deleted the wrong file, or the file was recreated before you noticed. The path is `cafe_concierge/.adk/session.db` (inside the `.adk` hidden subdirectory).
>
> **Fix:**
> ```bash
> # Make sure adk web is stopped first (Ctrl+C)
> # Delete the correct path
> rm -f cafe_concierge/.adk/session.db
>
> # Verify it's gone
> ls -la cafe_concierge/.adk/
>
> # Restart
> uv run adk web
> ```

#### Issue: Port 8000 in use after stopping and restarting

> **Symptom:** After Ctrl+C and restarting `uv run adk web`, the port is still in use.
>
> **Cause:** The previous process didn't fully terminate.
>
> **Fix:**
> ```bash
> lsof -ti:8000 | xargs kill -9 2>/dev/null || fuser -k 8000/tcp 2>/dev/null
> uv run adk web
> ```

---

### Step 7: Revisit Database Setup

#### Issue: Cloud SQL instance is not in `RUNNABLE` state

> **Symptom:** `gcloud sql instances describe cafe-concierge-db --format="value(state)"` returns `PENDING_CREATE` or `FAILED`.
>
> **Cause:** The instance creation from Step 2 is still in progress, or it failed in the background.
>
> **Fix:**
> - If `PENDING_CREATE`: Wait and check again. Instance creation takes 5-7 minutes.
>   ```bash
>   # Poll until ready
>   watch -n 10 'gcloud sql instances describe cafe-concierge-db --format="value(state)"'
>   ```
>   Press Ctrl+C once it shows `RUNNABLE`.
> - If `FAILED` or `SUSPENDED`: Delete and recreate:
>   ```bash
>   gcloud sql instances delete cafe-concierge-db --quiet
>   source .env
>   gcloud sql instances create cafe-concierge-db \
>     --database-version=POSTGRES_17 \
>     --edition=ENTERPRISE \
>     --region=${REGION} \
>     --availability-type=ZONAL \
>     --project=${GOOGLE_CLOUD_PROJECT} \
>     --tier=db-f1-micro \
>     --root-password=${DB_PASSWORD} \
>     --quiet
>   ```
>   (Remove the `&` this time so you can see when it finishes.)

#### Issue: Instance not found

> **Symptom:** `gcloud sql instances describe cafe-concierge-db` returns `NOT_FOUND` or `The Cloud SQL instance does not exist.`
>
> **Cause:** The background creation command from Step 2 failed silently, or was never run.
>
> **Fix:** Go back and run the creation command from Step 2 (this time without `&` so you can watch for errors):
> ```bash
> source .env
> gcloud sql instances create cafe-concierge-db \
>   --database-version=POSTGRES_17 \
>   --edition=ENTERPRISE \
>   --region=${REGION} \
>   --availability-type=ZONAL \
>   --project=${GOOGLE_CLOUD_PROJECT} \
>   --tier=db-f1-micro \
>   --root-password=${DB_PASSWORD} \
>   --quiet
> ```

#### Issue: `gcloud sql databases create` fails

> **Symptom:** `ERROR: (gcloud.sql.databases.create) INVALID_ARGUMENT` or permission errors when creating `agent_db`.
>
> **Cause:** The instance isn't fully ready, or the database name conflicts with an existing one.
>
> **Fix:**
> ```bash
> # Check instance state first
> gcloud sql instances describe cafe-concierge-db --format="value(state)"
>
> # If RUNNABLE, check if the database already exists
> gcloud sql databases list --instance=cafe-concierge-db
>
> # If agent_db is already listed, skip this step
> # If not, retry the creation
> gcloud sql databases create agent_db --instance=cafe-concierge-db
> ```

#### Issue: Cloud SQL Auth Proxy fails to start

> **Symptom:** `cloud-sql-proxy` returns `error: unable to connect` or `failed to dial` errors.
>
> **Cause:** The instance connection string is wrong, or the proxy can't authenticate.
>
> **Fix:**
> ```bash
> # Verify environment variables are set
> echo "Project: $GOOGLE_CLOUD_PROJECT"
> echo "Region: $REGION"
>
> # The connection string format is: PROJECT:REGION:INSTANCE_NAME
> cloud-sql-proxy ${GOOGLE_CLOUD_PROJECT}:${REGION}:cafe-concierge-db --port 5432 &
> ```
> If it still fails, verify your authentication:
> ```bash
> gcloud auth application-default login
> ```

#### Issue: `psql` connection test fails

> **Symptom:** The `psql` command to test the connection returns `connection refused` or `authentication failed`.
>
> **Cause:** The Cloud SQL Auth Proxy isn't running, or the password is wrong.
>
> **Fix:**
> 1. Check if the proxy is running:
>    ```bash
>    ss -tlnp | grep ':5432 '
>    ```
> 2. If not running, start it:
>    ```bash
>    source .env
>    cloud-sql-proxy ${GOOGLE_CLOUD_PROJECT}:${REGION}:cafe-concierge-db --port 5432 &
>    ```
> 3. If running but authentication fails, verify the password:
>    ```bash
>    echo $DB_PASSWORD  # Should print: cafe-agent-pwd-2025
>    ```
>    If blank: `source .env`

---

### Step 8: Verify Persistent Memory Across Sessions

#### Issue: Proxy restart command in the codelab has wrong syntax

> **Symptom:** The `if ss ... else ... fi` snippet from the codelab tries to start the proxy but fails with an invalid connection string error. The connection string resolves to something like `your-project:cafe-concierge-db` (missing the region).
>
> **Cause:** The codelab contains a typo in this step: `{$REGION}` should be `${REGION}`. The braces are reversed, so bash interprets `{$REGION}` as a literal `{us-central1}` with braces, which breaks the connection string format.
>
> **Fix:** Use this corrected version:
> ```bash
> if ss -tlnp | grep -q ':5432 '; then
>   echo "Cloud SQL Auth Proxy is already running."
> else
>   cloud-sql-proxy ${GOOGLE_CLOUD_PROJECT}:${REGION}:cafe-concierge-db --port 5432 &
> fi
> ```

#### Issue: `asyncpg` connection error when starting the agent with database URI

> **Symptom:** `ModuleNotFoundError: No module named 'asyncpg'` or connection errors when running the `uv run adk web --session_service_uri postgresql+asyncpg://...` command.
>
> **Cause:** The `asyncpg` package wasn't installed (should have been added in Step 3 with `uv add`).
>
> **Fix:**
> ```bash
> uv add asyncpg
> uv run adk web --session_service_uri postgresql+asyncpg://postgres:${DB_PASSWORD}@127.0.0.1:5432/agent_db
> ```

#### Issue: Database session creation takes too long or errors out

> **Symptom:** After the ADK dev UI loads, the session ID area shows loading for a long time, or an error appears in the chat when you send the first message.
>
> **Cause:** The first connection to the database takes extra time while ADK creates the required tables (`sessions`, `events`, `user_states`, `app_states`, `adk_internal_metadata`). Network latency between Cloud Shell and the Cloud SQL instance adds to the delay.
>
> **Fix:**
> 1. Wait 15-30 seconds after the session ID appears at the top of the UI before sending messages.
> 2. If it keeps failing, click the **New Session** button to manually create a new session.
> 3. If that also fails, check the terminal running `adk web` for error messages — common causes:
>    - Cloud SQL Auth Proxy not running (see [Quick Fix #4](#4-cloud-sql-auth-proxy-not-running))
>    - Wrong database password (verify with `echo $DB_PASSWORD`)

#### Issue: Agent is very slow to respond when connected to the database

> **Symptom:** Each agent response takes 10-30 seconds when connected to PostgreSQL, compared to 2-5 seconds with local storage.
>
> **Cause:** The agent is running locally in Cloud Shell while the Cloud SQL instance may be in a different zone. Every state read/write requires a network round-trip through the Cloud SQL Auth Proxy.
>
> **Fix:** This is expected behavior in this codelab setup. In a production deployment, the agent would run on Cloud Run in the same region as the database, eliminating this latency. No action needed — just be patient with responses.

#### Issue: Preferences don't persist after restart (Test 2 fails)

> **Symptom:** After stopping the agent, deleting `session.db`, and restarting with `--session_service_uri`, the agent doesn't remember your dietary preferences.
>
> **Cause:** Possible reasons:
> 1. The agent wasn't connected to the database during Test 1 (the `--session_service_uri` flag was missing).
> 2. The `user:` prefix is missing from the preference key in the code.
> 3. You're testing with a different user ID.
>
> **Fix:**
> 1. Check the terminal where `adk web` is running — verify it shows `DatabaseSessionService` in the startup logs (not `InMemorySessionService`).
> 2. Verify the `--session_service_uri` flag is present in both the Test 1 and Test 2 startup commands.
> 3. Query the database directly to check if data was saved:
>    ```bash
>    psql "host=127.0.0.1 port=5432 dbname=agent_db user=postgres password=$DB_PASSWORD" \
>      -c "SELECT * FROM user_states;"
>    ```

#### Issue: `psql` command not found when inspecting the database

> **Symptom:** `command not found: psql` when trying to query the database tables.
>
> **Cause:** The `psql` client may not be installed in your Cloud Shell instance.
>
> **Fix:**
> ```bash
> sudo apt-get update && sudo apt-get install -y postgresql-client
> ```
> Then retry the `psql` command.

#### Issue: New terminal tab can't connect to database

> **Symptom:** Opening a new terminal tab to run `psql` (as instructed in "Inspect the database directly") fails with connection refused.
>
> **Cause:** The Cloud SQL Auth Proxy is running in a different terminal tab. The proxy is accessible from any tab since it binds to `127.0.0.1:5432`, but the environment variables (`$DB_PASSWORD`) may not be set in the new tab.
>
> **Fix:**
> ```bash
> cd ~/build-agent-adk-cloudsql
> source .env
> psql "host=127.0.0.1 port=5432 dbname=agent_db user=postgres password=$DB_PASSWORD" -c "\dt"
> ```

---

### Step 9: Congratulations / Clean Up

#### Issue: `gcloud projects delete` asks for confirmation

> **Symptom:** The command prompts "Do you want to continue?" and waits for input.
>
> **Cause:** The `--quiet` flag wasn't used.
>
> **Fix:** Type `Y` and press Enter to confirm. Or re-run with `--quiet`:
> ```bash
> gcloud projects delete ${GOOGLE_CLOUD_PROJECT} --quiet
> ```

#### Issue: Cloud SQL instance deletion fails

> **Symptom:** `gcloud sql instances delete cafe-concierge-db` returns an error or takes a long time.
>
> **Cause:** The instance may have an active connection (Cloud SQL Auth Proxy) preventing deletion.
>
> **Fix:**
> ```bash
> # Kill the proxy first
> lsof -ti:5432 | xargs kill -9 2>/dev/null || fuser -k 5432/tcp 2>/dev/null
>
> # Then delete
> gcloud sql instances delete cafe-concierge-db --quiet
> ```

---

## General FAQ

### Q: Which Google account should I use — personal Gmail or Workspace?

**A:** For trial credits, use your **personal Gmail** account. Workspace accounts may have organizational policies that restrict project creation or billing. If you're in an instructor-led workshop, follow the instructor's guidance.

### Q: Can I use a different Gemini model instead of `gemini-2.5-flash`?

**A:** Yes, but you'd need to modify both the `adk create` command (Step 3) and the `model` parameter in the `root_agent` definition. Use the same model name in both places. Note that model availability varies by region.

### Q: What's the difference between `GOOGLE_CLOUD_LOCATION` and `REGION`?

**A:** `GOOGLE_CLOUD_LOCATION` (set to `global`) is used only for Vertex AI / Gemini API calls. `REGION` (set to `us-central1`) is used for all other GCP products like Cloud SQL, Cloud Run, and Artifact Registry. These are separate because Vertex AI and infrastructure resources have different regional availability.

### Q: Do I need to keep the Cloud Shell tab open the entire time?

**A:** Yes, while the agent is running (`uv run adk web`), the Cloud Shell terminal must remain open. If the session times out, the agent process stops. The Cloud SQL instance continues running independently. See [Environment Recovery](#environment-recovery) to resume.

### Q: Can I use a different database password?

**A:** Yes. Change the `DB_PASSWORD` value in your `.env` file and make sure to use the same password when creating the Cloud SQL instance. If you already created the instance with `cafe-agent-pwd-2025`, you'd need to reset the password:
```bash
gcloud sql users set-password postgres --instance=cafe-concierge-db --password=YOUR_NEW_PASSWORD
```

### Q: Why does the codelab use `uv run adk web` instead of just `adk web`?

**A:** `uv run` executes the command within the project's virtual environment managed by `uv`. This ensures the correct Python version and dependencies are used without manually activating a virtual environment.

### Q: I accidentally deleted my working directory. Can I recover?

**A:** The working directory (`~/build-agent-adk-cloudsql`) is in Cloud Shell's persistent home directory, so it survives VM restarts. But if you manually deleted it, you need to restart from Step 1. Your Cloud SQL instance and database remain intact — you only need to recreate the local files and reconnect.

---

## Environment Recovery

Use this section if you've been disconnected from Cloud Shell, your session timed out, or you need to get back to a known-good state.

### Recovering from Cloud Shell Disconnection

If your Cloud Shell session was disconnected or timed out:

1. **Re-open Cloud Shell** at [https://ide.cloud.google.com](https://ide.cloud.google.com). Click **View > Terminal** to open the terminal.

2. **Navigate to your working directory:**
   ```bash
   cd ~/build-agent-adk-cloudsql
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

5. **Restart the Cloud SQL Auth Proxy** (if you were on Step 7 or later):
   ```bash
   # Kill any stale proxy processes
   lsof -ti:5432 | xargs kill -9 2>/dev/null || fuser -k 5432/tcp 2>/dev/null

   # Restart the proxy
   cloud-sql-proxy ${GOOGLE_CLOUD_PROJECT}:${REGION}:cafe-concierge-db --port 5432 &
   ```

6. **Kill stale processes on port 8000** (if you were running `adk web`):
   ```bash
   lsof -ti:8000 | xargs kill -9 2>/dev/null || fuser -k 8000/tcp 2>/dev/null
   ```

7. **Resume where you left off** — your files, Cloud SQL instance, and database all persist across Cloud Shell reconnections. Only background processes (proxy, `adk web`) need to be restarted.

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
gcloud sql instances describe cafe-concierge-db --format="value(state)" 2>/dev/null || echo "Instance NOT FOUND"

# Check if Cloud SQL Auth Proxy is running
ss -tlnp | grep ':5432 ' && echo "Proxy: RUNNING" || echo "Proxy: NOT RUNNING"

# Check if adk web is running
ss -tlnp | grep ':8000 ' && echo "ADK Web: RUNNING" || echo "ADK Web: NOT RUNNING"

# Verify agent files exist
ls cafe_concierge/agent.py 2>/dev/null && echo "Agent code: EXISTS" || echo "Agent code: MISSING"
```

### Starting Over from a Specific Step

If you need to restart from a particular step:

- **From Step 1 (Set Up Your Environment):**
  ```bash
  cd ~
  rm -rf build-agent-adk-cloudsql
  mkdir -p ~/build-agent-adk-cloudsql
  cloudshell workspace ~/build-agent-adk-cloudsql && cd ~/build-agent-adk-cloudsql
  ```
  Then follow Step 1 from the beginning.

- **From Step 2 (Set Up Cloud SQL):**
  ```bash
  cd ~/build-agent-adk-cloudsql
  source .env
  # Check if instance already exists
  gcloud sql instances describe cafe-concierge-db --format="value(state)" 2>/dev/null
  # If it exists and is RUNNABLE, skip to Step 3
  # If not, re-run the creation command
  ```

- **From Step 3 (Build the Cafe Concierge Agent):**
  ```bash
  cd ~/build-agent-adk-cloudsql
  source .env
  rm -rf cafe_concierge .venv pyproject.toml uv.lock
  uv init
  uv add google-adk==1.25.0 asyncpg
  uv run adk create cafe_concierge \
      --model gemini-2.5-flash \
      --project ${GOOGLE_CLOUD_PROJECT} \
      --region ${GOOGLE_CLOUD_LOCATION}
  ```
  Then continue writing `agent.py` as described in Step 3.

- **From Step 7 (Revisit Database Setup):**
  ```bash
  cd ~/build-agent-adk-cloudsql
  bash setup_verify_trial_project.sh && source .env
  # Verify Cloud SQL instance
  gcloud sql instances describe cafe-concierge-db --format="value(state)"
  # If RUNNABLE, continue with creating the database and proxy
  ```

- **From Step 8 (Verify Persistent Memory):**
  ```bash
  cd ~/build-agent-adk-cloudsql
  bash setup_verify_trial_project.sh && source .env
  # Kill stale processes
  lsof -ti:8000 | xargs kill -9 2>/dev/null || fuser -k 8000/tcp 2>/dev/null
  lsof -ti:5432 | xargs kill -9 2>/dev/null || fuser -k 5432/tcp 2>/dev/null
  # Restart proxy
  cloud-sql-proxy ${GOOGLE_CLOUD_PROJECT}:${REGION}:cafe-concierge-db --port 5432 &
  # Wait for proxy to be ready, then start agent with database
  uv run adk web --session_service_uri postgresql+asyncpg://postgres:${DB_PASSWORD}@127.0.0.1:5432/agent_db
  ```

- **Complete restart (nuclear option):**
  ```bash
  cd ~
  rm -rf build-agent-adk-cloudsql
  # Optionally delete the Cloud SQL instance too
  gcloud sql instances delete cafe-concierge-db --quiet 2>/dev/null
  ```
  Then start from Step 1 of the codelab.
