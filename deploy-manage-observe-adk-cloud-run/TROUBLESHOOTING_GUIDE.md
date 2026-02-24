# Troubleshooting Guide: Deploy, Manage, and Observe ADK Agent on Cloud Run

> **Codelab URL:** https://codelabs.developers.google.com/deploy-manage-observe-adk-cloud-run
>
> **Last updated:** 2026-02-24

This guide helps you diagnose and fix common issues encountered while following the **Deploy, Manage, and Observe ADK Agent on Cloud Run** codelab. If you're stuck, start with the [Quick Fixes](#quick-fixes) section — it covers the most frequent problems.

---

## Table of Contents

- [Quick Fixes](#quick-fixes)
- [Step-by-Step Troubleshooting](#step-by-step-troubleshooting)
  - [Step 1: Preparing Workshop Setup](#step-1-preparing-workshop-setup)
  - [Step 2: Enabling APIs](#step-2-enabling-apis)
  - [Step 3: Python Environment Setup and Environment Variables](#step-3-python-environment-setup-and-environment-variables)
  - [Step 4: Preparing CloudSQL Database](#step-4-preparing-cloudsql-database)
  - [Step 5: Build the Weather Agent with ADK and Gemini 2.5](#step-5-build-the-weather-agent-with-adk-and-gemini-25)
  - [Step 6: Deploying to Cloud Run](#step-6-deploying-to-cloud-run)
  - [Step 7: The Dockerfile and Backend Server Script](#step-7-the-dockerfile-and-backend-server-script)
  - [Step 8: Inspecting Cloud Run Auto Scaling with Load Testing](#step-8-inspecting-cloud-run-auto-scaling-with-load-testing)
  - [Step 9: Gradual Release New Revisions](#step-9-gradual-release-new-revisions)
  - [Step 10: ADK Tracing](#step-10-adk-tracing)
  - [Step 12: Clean Up](#step-12-clean-up)
- [General FAQ](#general-faq)
- [Environment Recovery](#environment-recovery)

---

## Quick Fixes

The most common issues across all steps. **Check these first** before diving into step-specific troubleshooting.

### 1. Environment variables are empty

> **Symptom:** Commands like `echo $GOOGLE_CLOUD_PROJECT` return blank. `gcloud` commands fail with `ERROR: (gcloud) The required property [project] is not currently set`.
>
> **Cause:** Cloud Shell session timed out and environment was reset, or you opened a new terminal tab that doesn't have `.env` sourced.
>
> **Fix:**
> ```bash
> cd ~/deploy_and_manage_adk
> source .env
> ```
> Verify with `echo $GOOGLE_CLOUD_PROJECT` — it should print your project ID.

### 2. Cloud Shell session disconnected

> **Symptom:** Cloud Shell terminal is unresponsive, shows "Reconnecting...", or you return to a fresh terminal without your environment variables or running processes.
>
> **Cause:** Cloud Shell timed out due to inactivity (default: 20 minutes idle) or a network interruption.
>
> **Fix:** See the [Environment Recovery](#environment-recovery) section for full recovery steps. The short version:
> ```bash
> cd ~/deploy_and_manage_adk
> bash setup_trial_project.sh && source .env
> ```

### 3. Port 8080 already in use

> **Symptom:** `ERROR: [Errno 98] Address already in use` or `ERROR: Could not bind to port 8080` when running `uv run adk web --port 8080`.
>
> **Cause:** A previous `adk web` process wasn't stopped with Ctrl+C, or is still running from a disconnected session.
>
> **Fix:**
> ```bash
> lsof -ti:8080 | xargs kill -9 2>/dev/null || fuser -k 8080/tcp 2>/dev/null
> ```
> Then re-run your command.

### 4. Cloud SQL instance creation killed or interrupted

> **Symptom:** The Cloud SQL instance creation command from Step 4 was interrupted (terminal closed, Ctrl+C pressed), and now re-running it returns `ERROR: Resource already exists` or the instance is stuck in `PENDING_CREATE`.
>
> **Cause:** The terminal was closed during the 5-7 minute creation process.
>
> **Fix:**
> ```bash
> # Check the instance state
> gcloud sql instances describe adk-deployment --format="value(state)" 2>/dev/null
> ```
> - If `RUNNABLE`: The instance is ready. Skip to setting the password:
>   ```bash
>   gcloud sql users set-password postgres --instance=adk-deployment --password=ADK-deployment123
>   ```
> - If `PENDING_CREATE`: Wait and re-check. Creation takes 5-7 minutes.
> - If `FAILED` or no instance found: Delete and recreate:
>   ```bash
>   gcloud sql instances delete adk-deployment --quiet 2>/dev/null
>   gcloud sql instances create adk-deployment \
>     --database-version=POSTGRES_17 \
>     --edition=ENTERPRISE \
>     --tier=db-g1-small \
>     --region=us-central1 \
>     --availability-type=ZONAL \
>     --project=${GOOGLE_CLOUD_PROJECT}
>   ```

### 5. `DB_CONNECTION_NAME` not set or incorrect in `.env`

> **Symptom:** The `deploy_to_cloudrun.sh` script fails with `Error: DB_CONNECTION_NAME is not set in .env file`, or the deployed service can't connect to the database.
>
> **Cause:** The `DB_CONNECTION_NAME` variable was not uncommented and filled in the `.env` file (Step 6), or the value format is wrong.
>
> **Fix:**
> ```bash
> # Get the correct connection name
> gcloud sql instances describe adk-deployment --format="value(connectionName)"
> ```
> Copy the output (format: `project-id:us-central1:adk-deployment`) and update your `.env` file:
> ```bash
> cloudshell edit .env
> ```
> Set `DB_CONNECTION_NAME=<the value from above>`, then re-source:
> ```bash
> source .env
> ```

### 6. Cloud Run deployment fails

> **Symptom:** `bash deploy_to_cloudrun.sh` fails with build errors, permission errors, or timeout.
>
> **Cause:** Missing APIs, wrong project configuration, or Docker build issues.
>
> **Fix:**
> ```bash
> # Verify project and env vars
> echo "Project: $GOOGLE_CLOUD_PROJECT"
> echo "DB Connection: $DB_CONNECTION_NAME"
>
> # Ensure APIs are enabled
> gcloud services enable run.googleapis.com cloudbuild.googleapis.com
>
> # Re-run setup script and deployment
> bash setup_trial_project.sh && source .env
> bash deploy_to_cloudrun.sh
> ```

### 7. Vertex AI API errors

> **Symptom:** Agent responds with `Vertex AI API has not been used in project` or `403 Permission Denied` when interacting with the agent locally or on Cloud Run.
>
> **Cause:** The `aiplatform.googleapis.com` API wasn't enabled in Step 2, or enablement hasn't propagated yet.
>
> **Fix:**
> ```bash
> gcloud services enable aiplatform.googleapis.com
> ```
> Wait for `Operation/... finished successfully` before retrying. API propagation can take 1-2 minutes.

### 8. `SERVICE_URL` variable is empty during load testing

> **Symptom:** The `locust` load test command fails because `$SERVICE_URL` is empty or unset.
>
> **Cause:** The `export SERVICE_URL=$(gcloud run services describe ...)` command was not run, or it returned blank because the service name or region is wrong.
>
> **Fix:**
> ```bash
> export SERVICE_URL=$(gcloud run services describe weather-agent \
>     --platform managed \
>     --region us-central1 \
>     --format 'value(status.url)')
> echo $SERVICE_URL
> ```
> If the echo returns blank, verify the service exists:
> ```bash
> gcloud run services list --region us-central1
> ```

---

## Step-by-Step Troubleshooting

### Step 1: Preparing Workshop Setup

#### Issue: Trial billing account redemption fails

> **Symptom:** The redemption portal shows an error, says the code is invalid, or the page doesn't load.
>
> **Cause:** You may be using a Google Workspace email instead of a personal Gmail, or the redemption code has expired.
>
> **Fix:**
> 1. Open an **incognito window** and log in with a **personal Gmail** account (not Workspace/corporate email).
> 2. If the code is expired, ask your workshop instructor for a new one.
> 3. If self-studying, create a project with your own billing account instead.

#### Issue: `git clone` fails

> **Symptom:** `git clone https://github.com/alphinside/deploy-and-manage-adk-service.git deploy_and_manage_adk` returns `fatal: repository not found` or a network error.
>
> **Cause:** The repository URL may have changed, or Cloud Shell has a temporary network issue.
>
> **Fix:**
> 1. Retry the command — transient network issues are common in Cloud Shell.
> 2. Verify the URL is correct by checking with your workshop instructor.
> 3. If the repo name has changed, check the codelab page for an updated URL.

#### Issue: `cloudshell workspace` command doesn't change the editor view

> **Symptom:** After running `cloudshell workspace ~/deploy_and_manage_adk && cd ~/deploy_and_manage_adk`, the Cloud Shell Editor file tree still shows a different directory.
>
> **Cause:** The `cloudshell workspace` command sometimes requires a page refresh to take effect in the editor.
>
> **Fix:**
> 1. Refresh the Cloud Shell browser tab.
> 2. If the editor still shows the wrong directory, click **File > Open Folder** in the Cloud Shell Editor and navigate to `~/deploy_and_manage_adk`.
> 3. The terminal `cd` command works independently — verify with `pwd`.

#### Issue: Setup script fails with "no trial billing account found"

> **Symptom:** `setup_trial_project.sh` exits with an error about no trial billing account.
>
> **Cause:** The trial billing account was not successfully linked in the redemption step, or you're logged into Cloud Shell with a different Google account.
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
> Then re-run: `bash setup_trial_project.sh && source .env`

#### Issue: Setup script hangs at project ID prompt

> **Symptom:** The script displays a suggested project ID and waits indefinitely.
>
> **Cause:** The script is waiting for you to press Enter to accept the suggested ID, or type a custom one.
>
> **Fix:** Press **Enter** to accept the suggested project ID. If you want a custom ID, type it and then press Enter.

---

### Step 2: Enabling APIs

#### Issue: API enablement fails with billing error

> **Symptom:** `ERROR: (gcloud.services.enable) FAILED_PRECONDITION: Billing account for project is not found.`
>
> **Cause:** No billing account is linked to the project.
>
> **Fix:**
> 1. Go to [Billing](https://console.cloud.google.com/billing/linkedaccount).
> 2. Link your trial billing account.
> 3. Retry:
> ```bash
> gcloud services enable aiplatform.googleapis.com \
>                        run.googleapis.com \
>                        cloudbuild.googleapis.com \
>                        cloudresourcemanager.googleapis.com \
>                        sqladmin.googleapis.com \
>                        compute.googleapis.com
> ```

#### Issue: API enablement takes a long time

> **Symptom:** The `gcloud services enable` command runs for several minutes without completing.
>
> **Cause:** Enabling multiple APIs in a single command can take 2-5 minutes, especially if the Compute Engine API needs to initialize a default service account.
>
> **Fix:** Wait patiently. Do not interrupt the command. You should see `Operation "operations/..." finished successfully.` when done. If it takes more than 10 minutes, press Ctrl+C and try enabling one API at a time:
> ```bash
> gcloud services enable aiplatform.googleapis.com
> gcloud services enable run.googleapis.com
> gcloud services enable cloudbuild.googleapis.com
> gcloud services enable cloudresourcemanager.googleapis.com
> gcloud services enable sqladmin.googleapis.com
> gcloud services enable compute.googleapis.com
> ```

---

### Step 3: Python Environment Setup and Environment Variables

#### Issue: `uv sync --frozen` fails

> **Symptom:** `uv sync --frozen` returns errors about missing `uv.lock` file or dependency resolution failures.
>
> **Cause:** The `uv.lock` file is missing from the cloned repo, or the clone was incomplete.
>
> **Fix:**
> ```bash
> # Verify you're in the correct directory
> pwd  # Should show ~/deploy_and_manage_adk
>
> # Check if uv.lock exists
> ls uv.lock pyproject.toml
>
> # If files are missing, re-clone
> cd ~
> rm -rf deploy_and_manage_adk
> git clone https://github.com/alphinside/deploy-and-manage-adk-service.git deploy_and_manage_adk
> cd ~/deploy_and_manage_adk
> uv sync --frozen
> ```

#### Issue: `.env` file not visible in Cloud Shell Editor

> **Symptom:** After running `cloudshell open .env`, the file doesn't appear in the editor. The file tree doesn't show it either.
>
> **Cause:** Files starting with `.` are hidden by default in the Cloud Shell Editor.
>
> **Fix:** In the Cloud Shell Editor, click **View > Toggle Hidden Files** to display hidden files. The `.env` file should then appear in the file tree.

#### Issue: `GOOGLE_CLOUD_PROJECT` shows `your-project-id` instead of actual project ID

> **Symptom:** `echo $GOOGLE_CLOUD_PROJECT` prints `your-project-id` literally, or the `.env` file shows placeholder values.
>
> **Cause:** The `setup_trial_project.sh` script didn't run successfully, or `.env` wasn't sourced after the script completed.
>
> **Fix:**
> ```bash
> bash setup_trial_project.sh && source .env
> echo $GOOGLE_CLOUD_PROJECT  # Should show your actual project ID
> ```

---

### Step 4: Preparing CloudSQL Database

#### Issue: Cloud SQL instance creation fails immediately

> **Symptom:** The `gcloud sql instances create` command returns an error about quotas, permissions, or the API not being enabled.
>
> **Cause:** The `sqladmin.googleapis.com` or `compute.googleapis.com` API wasn't enabled in Step 2, or your project has hit a resource quota.
>
> **Fix:**
> ```bash
> gcloud services enable sqladmin.googleapis.com compute.googleapis.com
>
> # Retry instance creation
> gcloud sql instances create adk-deployment \
>   --database-version=POSTGRES_17 \
>   --edition=ENTERPRISE \
>   --tier=db-g1-small \
>   --region=us-central1 \
>   --availability-type=ZONAL \
>   --project=${GOOGLE_CLOUD_PROJECT}
> ```

#### Issue: Terminal closed or Ctrl+C pressed during instance creation

> **Symptom:** The terminal was accidentally closed or you pressed Ctrl+C during the 5-7 minute instance creation. You're unsure if the instance was created.
>
> **Cause:** Interrupting the `gcloud` command doesn't stop the server-side operation — the instance may still be creating.
>
> **Fix:**
> ```bash
> # Check if the instance exists and its state
> gcloud sql instances describe adk-deployment --format="value(state)" 2>/dev/null
> ```
> - `RUNNABLE`: Instance is ready. Proceed to set the password:
>   ```bash
>   gcloud sql users set-password postgres --instance=adk-deployment --password=ADK-deployment123
>   ```
> - `PENDING_CREATE`: Wait and re-check in a few minutes.
> - No output / `NOT_FOUND`: Re-run the creation command from Step 4.

#### Issue: Instance already exists

> **Symptom:** `ERROR: (gcloud.sql.instances.create) Resource already exists` when re-running the creation command.
>
> **Cause:** The instance was already created (possibly from a previous attempt or a background operation completed).
>
> **Fix:**
> ```bash
> gcloud sql instances describe adk-deployment --format="value(state)"
> ```
> If it returns `RUNNABLE`, the instance is ready — proceed to the password step. If it's in a bad state:
> ```bash
> gcloud sql instances delete adk-deployment --quiet
> # Then re-run the creation command from Step 4
> ```

#### Issue: Password set command fails

> **Symptom:** `gcloud sql users set-password postgres` fails with an error about the instance not being ready or not found.
>
> **Cause:** The instance creation (first command in the chained `&&` command) failed, so the password command didn't execute. Or the instance is still in `PENDING_CREATE` state.
>
> **Fix:**
> ```bash
> # Wait for instance to be ready
> gcloud sql instances describe adk-deployment --format="value(state)"
> # Should show RUNNABLE
>
> # Then set the password
> gcloud sql users set-password postgres --instance=adk-deployment --password=ADK-deployment123
> ```

---

### Step 5: Build the Weather Agent with ADK and Gemini 2.5

#### Issue: `uv run adk web --port 8080` fails with "command not found"

> **Symptom:** `error: No such command: adk` or `uv run adk: command not found`.
>
> **Cause:** Dependencies weren't installed (Step 3's `uv sync --frozen` was skipped or failed).
>
> **Fix:**
> ```bash
> cd ~/deploy_and_manage_adk
> uv sync --frozen
> uv run adk web --port 8080
> ```

#### Issue: Web Preview shows blank page or "Unable to connect"

> **Symptom:** After running `uv run adk web --port 8080` and opening Web Preview on port 8080, the page is blank or shows a connection error.
>
> **Cause:** The server hasn't fully started, or the wrong port is selected in Web Preview.
>
> **Fix:**
> 1. Check the terminal — wait until you see `Uvicorn running on http://0.0.0.0:8080`.
> 2. In Web Preview, confirm the port is set to **8080** (click the Web Preview button, then "Preview on port 8080").
> 3. Wait a few seconds and refresh the browser tab.
> 4. Select `weather_agent` from the agent dropdown in the top-left corner of the dev UI.

#### Issue: Web Preview button not visible

> **Symptom:** You can't find the Web Preview button in Cloud Shell.
>
> **Cause:** The button is in the top-right area of the Cloud Shell Editor toolbar and may not be immediately obvious.
>
> **Fix:** Look for a button with a square-and-arrow icon in the **top toolbar** of Cloud Shell Editor. It's typically in the upper-right area. Click it and select **Preview on port 8080**. If you still can't find it, try the keyboard shortcut or look under **View > Web Preview**.

#### Issue: Agent returns errors or doesn't respond

> **Symptom:** After selecting `weather_agent` and typing a message, the chat returns an error or hangs indefinitely.
>
> **Cause:** Vertex AI API issues, wrong project ID in the `.env`, or the model endpoint is not reachable.
>
> **Fix:**
> ```bash
> # Verify environment is set
> echo $GOOGLE_CLOUD_PROJECT
> echo $GOOGLE_CLOUD_LOCATION  # Should be "global"
> echo $GOOGLE_GENAI_USE_VERTEXAI  # Should be "True"
>
> # Verify API is enabled
> gcloud services list --enabled --filter="name:aiplatform.googleapis.com"
> ```
> If `GOOGLE_CLOUD_PROJECT` is blank, re-source your environment: `source .env`
> If the API is not listed, enable it: `gcloud services enable aiplatform.googleapis.com`

#### Issue: Forgetting to kill the `adk web` process before moving on

> **Symptom:** Later steps fail because port 8080 is occupied, or you're confused about which process is running.
>
> **Cause:** The codelab reminds you to push **CTRL + C** to kill the process, but it's easy to miss.
>
> **Fix:** Before proceeding to Step 6, make sure you pressed **Ctrl+C** in the terminal running `adk web`. Verify:
> ```bash
> lsof -ti:8080 | xargs kill -9 2>/dev/null || fuser -k 8080/tcp 2>/dev/null
> ```

---

### Step 6: Deploying to Cloud Run

#### Issue: `bash setup_trial_project.sh && source .env` fails or resets environment

> **Symptom:** Running the setup script again changes your project or environment variables unexpectedly.
>
> **Cause:** The setup script creates a new project if it can't find the existing one, or you're logged in with a different account.
>
> **Fix:**
> ```bash
> # Check your current project
> gcloud config get-value project
>
> # If the project is correct, just source .env
> source .env
>
> # If you need to reset to the correct project
> gcloud config set project YOUR_PROJECT_ID
> source .env
> ```

#### Issue: Can't find `DB_CONNECTION_NAME` in Cloud Console

> **Symptom:** You navigated to the Cloud SQL dashboard but can't locate the "Connection Name" value for the `adk-deployment` instance.
>
> **Cause:** The connection name is in the instance detail page, in the "Connect to this instance" section, which requires scrolling down.
>
> **Fix:**
> You can get the connection name from the terminal instead of the console:
> ```bash
> gcloud sql instances describe adk-deployment --format="value(connectionName)"
> ```
> Copy the output value (format: `project-id:us-central1:adk-deployment`) and paste it as the `DB_CONNECTION_NAME` value in your `.env` file.

#### Issue: `.env` file still has placeholder value for `DB_CONNECTION_NAME`

> **Symptom:** The `deploy_to_cloudrun.sh` script fails because `DB_CONNECTION_NAME` is still set to `your-db-connection-name` or is commented out.
>
> **Cause:** The `.env` file wasn't updated with the actual connection name value.
>
> **Fix:**
> ```bash
> # Get the value
> CONNECTION_NAME=$(gcloud sql instances describe adk-deployment --format="value(connectionName)")
> echo "Connection name: $CONNECTION_NAME"
>
> # Open .env to edit
> cloudshell edit .env
> ```
> Replace `your-db-connection-name` with the actual connection name value. Save the file, then:
> ```bash
> source .env
> echo $DB_CONNECTION_NAME  # Should show project-id:us-central1:adk-deployment
> ```

#### Issue: Artifact registry confirmation prompt

> **Symptom:** The deployment script pauses and asks `Deploying from source requires an Artifact Registry Docker repository to store built containers. A repository named [cloud-run-source-deploy] in region [us-central1] will be created. Do you want to continue (Y/n)?`
>
> **Cause:** This is the first time deploying from source in this project, so Cloud Run needs to create a Docker repository in Artifact Registry.
>
> **Fix:** Type **Y** and press Enter. This is expected and only happens once.

#### Issue: Cloud Build fails during deployment

> **Symptom:** The deployment fails with `ERROR: build step 0 ... failed` or Docker build errors in the output.
>
> **Cause:** The `cloudbuild.googleapis.com` API may not be enabled, or there's a syntax error in the Dockerfile or project files.
>
> **Fix:**
> ```bash
> # Ensure Cloud Build API is enabled
> gcloud services enable cloudbuild.googleapis.com
>
> # Verify you're in the correct directory
> pwd  # Should be ~/deploy_and_manage_adk
>
> # Verify Dockerfile exists
> ls Dockerfile
>
> # Retry deployment
> bash deploy_to_cloudrun.sh
> ```

#### Issue: Deployment takes very long

> **Symptom:** The `deploy_to_cloudrun.sh` script runs for 10+ minutes without completing.
>
> **Cause:** The first deployment includes building the Docker image in Cloud Build, pushing to Artifact Registry, and provisioning the Cloud Run service. This is normal.
>
> **Fix:** Wait patiently. Do not close the terminal. The first deployment typically takes 5-10 minutes. You'll see a service URL printed when it completes. While waiting, proceed to read Step 7 (Dockerfile explanation).

#### Issue: Deployed service returns 500 errors

> **Symptom:** Accessing the Cloud Run URL shows an error page or the agent doesn't respond.
>
> **Cause:** The `SESSION_SERVICE_URI` environment variable has an incorrect database connection string, or the Cloud SQL instance isn't accessible from Cloud Run.
>
> **Fix:**
> ```bash
> # Check the service logs
> gcloud run services logs read weather-agent --region us-central1 --limit 20
>
> # Verify the Cloud SQL instance connection name
> gcloud sql instances describe adk-deployment --format="value(connectionName)"
>
> # Verify the deployed env vars
> gcloud run services describe weather-agent --region us-central1 \
>     --format="value(spec.template.spec.containers[0].env)"
> ```
> If the `SESSION_SERVICE_URI` or `DB_CONNECTION_NAME` is wrong, fix `.env` and re-deploy:
> ```bash
> source .env
> bash deploy_to_cloudrun.sh
> ```

---

### Step 7: The Dockerfile and Backend Server Script

#### Issue: Can't access deployed service from browser

> **Symptom:** Opening the Cloud Run URL in a browser shows "Error: Page not found" or a blank page, not the ADK dev UI.
>
> **Cause:** The service may still be starting up (cold start), or the URL is incorrect.
>
> **Fix:**
> 1. Wait 10-15 seconds and refresh — the first request triggers a cold start.
> 2. Verify the URL is correct:
>    ```bash
>    gcloud run services describe weather-agent --region us-central1 --format="value(status.url)"
>    ```
> 3. Open the URL in an **incognito window** to avoid browser cache issues.

---

### Step 8: Inspecting Cloud Run Auto Scaling with Load Testing

#### Issue: `SERVICE_URL` variable is empty

> **Symptom:** `echo $SERVICE_URL` prints nothing, and the locust command fails with an empty host.
>
> **Cause:** The `export SERVICE_URL=...` command was not run, or it returned blank because the service doesn't exist or the region is wrong.
>
> **Fix:**
> ```bash
> # Verify the service exists
> gcloud run services list --region us-central1
>
> # Get the URL
> export SERVICE_URL=$(gcloud run services describe weather-agent \
>     --platform managed \
>     --region us-central1 \
>     --format 'value(status.url)')
> echo "Service URL: $SERVICE_URL"
> ```
> If the service isn't listed, go back to Step 6 and deploy.

#### Issue: `locust` command not found

> **Symptom:** `uv run locust` returns `error: No such command: locust` or `ModuleNotFoundError`.
>
> **Cause:** The `locust` package is not listed in the project dependencies, or `uv sync` wasn't run after cloning.
>
> **Fix:**
> ```bash
> cd ~/deploy_and_manage_adk
> uv sync --frozen
> uv run locust -f load_test.py -H $SERVICE_URL -u 60 -r 5 -t 120 --headless
> ```
> If locust is still not found:
> ```bash
> uv add locust
> uv run locust -f load_test.py -H $SERVICE_URL -u 60 -r 5 -t 120 --headless
> ```

#### Issue: Load test shows high failure rate

> **Symptom:** The locust output shows a high percentage of failures (e.g., `# fails 50(80.00%)`).
>
> **Cause:** The Cloud Run service is returning errors, the Cloud SQL connection pool is exhausted, or the Vertex AI API quota is being hit.
>
> **Fix:**
> 1. Check the Cloud Run logs for error details:
>    ```bash
>    gcloud run services logs read weather-agent --region us-central1 --limit 50
>    ```
> 2. If you see `429 Resource Exhausted` errors, the Gemini API quota is being hit. Reduce the load test parameters:
>    ```bash
>    uv run locust -f load_test.py -H $SERVICE_URL -u 20 -r 2 -t 60 --headless
>    ```
> 3. If you see database connection errors, the Cloud SQL instance may be overloaded with the `db-g1-small` tier.

#### Issue: Load test runs but Cloud Run dashboard doesn't show scaling

> **Symptom:** After the load test, the Cloud Run Metrics tab doesn't show any auto-scaling activity (instance count stays at 1).
>
> **Cause:** The dashboard may not have refreshed, or the time range filter is wrong.
>
> **Fix:**
> 1. In the Cloud Run service detail page, click the **Metrics** tab.
> 2. Set the time range to **Last 15 minutes** or **Last 30 minutes**.
> 3. Look for the **Container instance count** chart — it should show scaling activity during the load test window.
> 4. Click the **Refresh** button if the data looks stale.

---

### Step 9: Gradual Release New Revisions

#### Issue: `cloudshell edit weather_agent/agent.py` doesn't open the file

> **Symptom:** The command runs but the editor doesn't open the file, or it opens a blank file.
>
> **Cause:** Cloud Shell Editor may have a delay in opening files, or the workspace path doesn't match.
>
> **Fix:** Try the alternative:
> ```bash
> cloudshell open weather_agent/agent.py
> ```
> Or navigate to the file manually in the Cloud Shell Editor file tree: expand `weather_agent/` folder and click `agent.py`.

#### Issue: Forgot to save the file before deploying

> **Symptom:** After deploying the new revision, the agent behavior hasn't changed — it still answers non-weather queries.
>
> **Cause:** The modified `agent.py` wasn't saved before running `gcloud run deploy`. The Cloud Shell Editor doesn't auto-save.
>
> **Fix:**
> 1. Open `weather_agent/agent.py` in the editor.
> 2. Press **Ctrl+S** to save.
> 3. Verify the file contains the updated instruction (`You only answer inquiries about the weather. Refuse all other user query`):
>    ```bash
>    grep "Refuse all other" weather_agent/agent.py
>    ```
> 4. Re-deploy:
>    ```bash
>    gcloud run deploy weather-agent \
>        --source . \
>        --port 8080 \
>        --project $GOOGLE_CLOUD_PROJECT \
>        --allow-unauthenticated \
>        --region us-central1 \
>        --no-traffic
>    ```

#### Issue: `GOOGLE_CLOUD_PROJECT` env var is empty during deploy

> **Symptom:** The `gcloud run deploy` command fails because `$GOOGLE_CLOUD_PROJECT` resolves to blank.
>
> **Cause:** The environment variable was lost due to a Cloud Shell reconnection or new terminal tab.
>
> **Fix:**
> ```bash
> cd ~/deploy_and_manage_adk
> source .env
> echo $GOOGLE_CLOUD_PROJECT
> # Now re-run the deploy command
> ```

#### Issue: New revision is serving traffic instead of 0%

> **Symptom:** After deploying with `--no-traffic`, the new revision is already receiving traffic.
>
> **Cause:** The `--no-traffic` flag was omitted from the deploy command, or you accidentally ran the `deploy_to_cloudrun.sh` script instead of the manual `gcloud run deploy` command with `--no-traffic`.
>
> **Fix:** You can still manage traffic manually via the console:
> 1. Go to [Cloud Run](https://console.cloud.google.com/run/services).
> 2. Click your `weather-agent` service.
> 3. Go to the **Revisions** tab.
> 4. Click the kebab menu (three dots) next to any revision and select **Manage Traffic**.
> 5. Adjust traffic percentages as desired.

#### Issue: Can't find the "Manage Traffic" option in Cloud Run console

> **Symptom:** You're on the Revisions tab but can't find the kebab button or "Manage Traffic" option.
>
> **Cause:** The UI layout may vary. The kebab button (three vertical dots) is at the end of each revision row.
>
> **Fix:**
> 1. In the **Revisions** tab, look for the three vertical dots (**...**) to the right of each revision entry.
> 2. Alternatively, click the **MANAGE TRAFFIC** button at the top of the Revisions tab (if available in your console version).
> 3. You can also manage traffic from the command line:
>    ```bash
>    # List revisions
>    gcloud run revisions list --service weather-agent --region us-central1
>
>    # Set traffic split (replace revision names with your actual revision names)
>    gcloud run services update-traffic weather-agent \
>        --region us-central1 \
>        --to-revisions=REVISION_1=50,REVISION_2=50
>    ```

---

### Step 10: ADK Tracing

#### Issue: No traces appear in Trace Explorer

> **Symptom:** After interacting with the deployed agent, the Trace Explorer page shows no spans or traces.
>
> **Cause:** Traces may take 1-2 minutes to appear. The filter might be wrong, or `trace_to_cloud=True` wasn't set in the server code.
>
> **Fix:**
> 1. Wait 2-3 minutes after interacting with the agent, then refresh the Trace Explorer page.
> 2. Make sure you're looking at the correct project (check the project selector in the Cloud Console header).
> 3. Adjust the time range to **Last 1 hour**.
> 4. In the **Span name** filter, search for `agent_run` to find spans specific to your agent.
> 5. Verify the `server.py` includes `"trace_to_cloud": True` in the `app_args` dict.

#### Issue: Trace Explorer page not loading

> **Symptom:** Navigating to the Trace Explorer shows an error or asks to enable an API.
>
> **Cause:** The Cloud Trace API may not be enabled.
>
> **Fix:**
> ```bash
> gcloud services enable cloudtrace.googleapis.com
> ```
> Then refresh the [Trace Explorer](https://console.cloud.google.com/traces/explorer) page.

#### Issue: Spans appear but don't show agent-specific details

> **Symptom:** Traces appear in the explorer, but they only show generic HTTP spans, not the `agent_run [weather_agent]` span with tool call details.
>
> **Cause:** You may be looking at infrastructure-level traces rather than agent-level traces.
>
> **Fix:** In the Trace Explorer:
> 1. Use the **Span name** filter and type `agent_run`.
> 2. Select the span named `agent_run [weather_agent]` from the filter results.
> 3. Click on individual traces in the list below to see the waterfall view with tool calls, LLM invocations, and timing details.

---

### Step 12: Clean Up

#### Issue: Project deletion confirmation prompt

> **Symptom:** Deleting the project asks for confirmation or the project ID to be typed.
>
> **Cause:** Google Cloud requires explicit confirmation to prevent accidental deletions.
>
> **Fix:** Follow the prompt — type the project ID when asked, then click **Shut down**. Alternatively, delete individual resources:
> ```bash
> # Delete Cloud Run service
> gcloud run services delete weather-agent --region us-central1 --quiet
>
> # Delete Cloud SQL instance
> gcloud sql instances delete adk-deployment --quiet
>
> # Delete Artifact Registry repository
> gcloud artifacts repositories delete cloud-run-source-deploy --location us-central1 --quiet
> ```

#### Issue: Resources still incurring charges after clean up

> **Symptom:** You receive billing alerts after completing the codelab.
>
> **Cause:** The Cloud SQL instance or Cloud Run service was not deleted, or the project itself was not shut down.
>
> **Fix:**
> ```bash
> # Check for running Cloud Run services
> gcloud run services list --region us-central1
>
> # Check for Cloud SQL instances
> gcloud sql instances list
>
> # Delete everything if still present
> gcloud run services delete weather-agent --region us-central1 --quiet 2>/dev/null
> gcloud sql instances delete adk-deployment --quiet 2>/dev/null
>
> # Or delete the entire project
> gcloud projects delete ${GOOGLE_CLOUD_PROJECT} --quiet
> ```

---

## General FAQ

### Q: Which Google account should I use — personal Gmail or Workspace?

**A:** For trial credits, use your **personal Gmail** account. Workspace accounts may have organizational policies that restrict project creation or billing. If you're in an instructor-led workshop, follow the instructor's guidance.

### Q: What is the database password used in this codelab?

**A:** The codelab uses `ADK-deployment123` as the password for the default `postgres` user. This is hardcoded in the deployment script and Cloud SQL setup commands. In production, you should use Secret Manager to store credentials securely.

### Q: Can I use a different Gemini model instead of `gemini-2.5-flash`?

**A:** Yes, but you'd need to modify the `model` parameter in `weather_agent/agent.py`. Use the same model name consistently. Note that model availability and latency varies — `gemini-2.5-flash` is optimized for speed, which matters for load testing and tracing steps.

### Q: What's the difference between `GOOGLE_CLOUD_LOCATION` and the region `us-central1`?

**A:** `GOOGLE_CLOUD_LOCATION` (set to `global`) is used for Vertex AI / Gemini API calls. The `us-central1` region is used for Cloud SQL, Cloud Run, and other infrastructure resources. These are separate because Vertex AI and infrastructure resources have different regional availability and endpoint configurations.

### Q: Do I need to keep the Cloud Shell tab open the entire time?

**A:** For steps running local processes (Step 5's `adk web`), yes. For deployed services on Cloud Run (Steps 6-10), no — Cloud Run manages the service independently. If Cloud Shell disconnects, see [Environment Recovery](#environment-recovery).

### Q: Why does the codelab use `uv run` instead of `python` directly?

**A:** `uv run` executes the command within the project's virtual environment managed by `uv`. This ensures the correct Python version and dependencies are used without manually activating a virtual environment.

### Q: The load test is slow. Can I reduce the number of users?

**A:** Yes. Reduce the `-u` flag (total users) and `-r` flag (ramp-up rate). For example:
```bash
uv run locust -f load_test.py -H $SERVICE_URL -u 20 -r 2 -t 60 --headless
```
You'll still see auto-scaling behavior, just at a smaller scale.

### Q: Can I use `--no-traffic` with the `deploy_to_cloudrun.sh` script?

**A:** No. The script doesn't accept flags — it uses the hardcoded `gcloud run deploy` command inside it. For the gradual release step (Step 9), use the manual `gcloud run deploy` command directly as shown in the codelab.

---

## Environment Recovery

Use this section if you've been disconnected from Cloud Shell, your session timed out, or you need to get back to a known-good state.

### Recovering from Cloud Shell Disconnection

If your Cloud Shell session was disconnected or timed out:

1. **Re-open Cloud Shell** at [https://shell.cloud.google.com](https://shell.cloud.google.com). Click **View > Terminal** to open the terminal.

2. **Navigate to your working directory:**
   ```bash
   cd ~/deploy_and_manage_adk
   ```

3. **Reload your environment variables and project configuration:**
   ```bash
   bash setup_trial_project.sh && source .env
   ```

4. **Verify key environment variables:**
   ```bash
   echo "Project: $GOOGLE_CLOUD_PROJECT"
   echo "Location: $GOOGLE_CLOUD_LOCATION"
   echo "DB Connection: $DB_CONNECTION_NAME"
   ```
   All values should be non-empty.

5. **Kill any stale processes** that may conflict:
   ```bash
   lsof -ti:8080 | xargs kill -9 2>/dev/null || fuser -k 8080/tcp 2>/dev/null
   ```

6. **Resume where you left off** — your files, Cloud SQL instance, Cloud Run service, and database all persist across Cloud Shell reconnections. Only local processes (`adk web`) need to be restarted.

### Verifying Your Environment State

Run these checks to confirm your environment is properly configured:

```bash
# Check project
echo "Project: $GOOGLE_CLOUD_PROJECT"

# Check location
echo "Location: $GOOGLE_CLOUD_LOCATION"

# Check DB connection name
echo "DB Connection: $DB_CONNECTION_NAME"

# Verify gcloud project
gcloud config get-value project

# Check if Cloud SQL instance exists and is running
gcloud sql instances describe adk-deployment --format="value(state)" 2>/dev/null || echo "Instance NOT FOUND"

# Check if Cloud Run service is deployed
gcloud run services describe weather-agent --region us-central1 --format="value(status.url)" 2>/dev/null || echo "Service NOT DEPLOYED"

# Verify agent files exist
ls weather_agent/agent.py 2>/dev/null && echo "Agent code: EXISTS" || echo "Agent code: MISSING"

# Check if locust test script exists
ls load_test.py 2>/dev/null && echo "Load test script: EXISTS" || echo "Load test script: MISSING"
```

### Starting Over from a Specific Step

If you need to restart from a particular step:

- **From Step 1 (Preparing Workshop Setup):**
  ```bash
  cd ~
  rm -rf deploy_and_manage_adk
  git clone https://github.com/alphinside/deploy-and-manage-adk-service.git deploy_and_manage_adk
  cloudshell workspace ~/deploy_and_manage_adk && cd ~/deploy_and_manage_adk
  bash setup_trial_project.sh && source .env
  ```

- **From Step 4 (Preparing CloudSQL Database):**
  ```bash
  cd ~/deploy_and_manage_adk
  source .env
  # Check if instance already exists
  gcloud sql instances describe adk-deployment --format="value(state)" 2>/dev/null
  # If RUNNABLE, skip to Step 5
  # If not found, re-run the creation command from Step 4
  ```

- **From Step 5 (Build the Weather Agent):**
  ```bash
  cd ~/deploy_and_manage_adk
  source .env
  uv sync --frozen
  # Kill any process on port 8080
  lsof -ti:8080 | xargs kill -9 2>/dev/null || fuser -k 8080/tcp 2>/dev/null
  uv run adk web --port 8080
  ```

- **From Step 6 (Deploying to Cloud Run):**
  ```bash
  cd ~/deploy_and_manage_adk
  bash setup_trial_project.sh && source .env
  # Ensure DB_CONNECTION_NAME is set
  echo $DB_CONNECTION_NAME
  # If empty, get it:
  gcloud sql instances describe adk-deployment --format="value(connectionName)"
  # Update .env with the value, then:
  source .env
  bash deploy_to_cloudrun.sh
  ```

- **From Step 8 (Load Testing):**
  ```bash
  cd ~/deploy_and_manage_adk
  source .env
  export SERVICE_URL=$(gcloud run services describe weather-agent \
      --platform managed \
      --region us-central1 \
      --format 'value(status.url)')
  echo "Service URL: $SERVICE_URL"
  uv run locust -f load_test.py -H $SERVICE_URL -u 60 -r 5 -t 120 --headless
  ```

- **From Step 9 (Gradual Release):**
  ```bash
  cd ~/deploy_and_manage_adk
  source .env
  # Edit agent.py if not already done
  grep "Refuse all other" weather_agent/agent.py || echo "NEED TO EDIT agent.py"
  # Deploy with no traffic
  gcloud run deploy weather-agent \
      --source . \
      --port 8080 \
      --project $GOOGLE_CLOUD_PROJECT \
      --allow-unauthenticated \
      --region us-central1 \
      --no-traffic
  ```

- **Complete restart (nuclear option):**
  ```bash
  cd ~
  rm -rf deploy_and_manage_adk
  # Optionally delete all cloud resources
  gcloud run services delete weather-agent --region us-central1 --quiet 2>/dev/null
  gcloud sql instances delete adk-deployment --quiet 2>/dev/null
  gcloud artifacts repositories delete cloud-run-source-deploy --location us-central1 --quiet 2>/dev/null
  ```
  Then start from Step 1 of the codelab.
