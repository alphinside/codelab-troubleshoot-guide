# Troubleshooting Guide: Building AI Agents with ADK: The Foundation

> **Codelab URL:** https://codelabs.developers.google.com/devsite/codelabs/build-agents-with-adk-foundation
>
> **Last updated:** 2026-02-24

This guide helps you diagnose and fix common issues encountered while following the **Building AI Agents with ADK: The Foundation** codelab. If you're stuck, start with the [Quick Fixes](#quick-fixes) section — it covers the most frequent problems.

---

## Table of Contents

- [Quick Fixes](#quick-fixes)
- [Step-by-Step Troubleshooting](#step-by-step-troubleshooting)
  - [Step 3: Configure Google Cloud Services](#step-3-configure-google-cloud-services)
  - [Step 4: Create a Python Virtual Environment](#step-4-create-a-python-virtual-environment)
  - [Step 5: Create an Agent](#step-5-create-an-agent)
  - [Step 6: Explore Agent Codes](#step-6-explore-agent-codes)
  - [Step 7: Run the Agent on the Terminal](#step-7-run-the-agent-on-the-terminal)
  - [Step 8: Run the Agent on the Development Web UI](#step-8-run-the-agent-on-the-development-web-ui)
- [General FAQ](#general-faq)
- [Environment Recovery](#environment-recovery)

---

## Quick Fixes

The most common issues across all steps. **Check these first** before diving into step-specific troubleshooting.

### 1. Virtual environment not activated

> **Symptom:** Running `adk run personal_assistant` or `adk web` returns `command not found: adk` or `ModuleNotFoundError: No module named 'google.adk'`.
>
> **Cause:** The Python virtual environment was deactivated — either because you opened a new terminal tab, Cloud Shell reconnected, or you never activated it.
>
> **Fix:**
> ```bash
> cd ~/ai-agents-adk
> source .venv/bin/activate
> ```
> You should see `(ai-agents-adk)` prefixing your terminal prompt.

### 2. Wrong project ID in `.env` file

> **Symptom:** Running the agent returns `{'message': 'This API method requires billing to be enabled'}` or `Permission denied` errors referencing a project you don't recognize.
>
> **Cause:** The `GOOGLE_CLOUD_PROJECT` value in `personal_assistant/.env` does not match your actual Google Cloud project ID.
>
> **Fix:**
> ```bash
> # Check your current project ID
> gcloud config get-value project
>
> # Verify the .env file matches
> cat ~/ai-agents-adk/personal_assistant/.env
> ```
> If they don't match, edit `personal_assistant/.env` and set `GOOGLE_CLOUD_PROJECT` to the correct project ID.

### 3. Vertex AI API not enabled

> **Symptom:** Error message containing `Vertex AI API has not been used in project` or `403 Permission Denied` when running the agent.
>
> **Cause:** The Vertex AI API was not enabled for your project, or the enablement hasn't propagated yet.
>
> **Fix:**
> ```bash
> gcloud services enable aiplatform.googleapis.com
> ```
> Wait for the `Operation/... finished successfully` message before retrying.

### 4. Billing not linked to the project

> **Symptom:** `{'message': 'This API method requires billing to be enabled'}` when running the agent.
>
> **Cause:** Your Google Cloud project does not have an active billing account linked.
>
> **Fix:**
> 1. Go to the [linked billing account page](https://console.cloud.google.com/billing/linkedaccount).
> 2. If no billing account is linked, select **Google Cloud Platform Trial Billing Account** from the dropdown.
> 3. Click **Set Account**.

### 5. Port already in use when running `adk web`

> **Symptom:** `ERROR: [Errno 98] Address already in use` or `ERROR: Could not bind to port 8000` when running `adk web`.
>
> **Cause:** A previous `adk web` or `adk run` process is still running in another terminal tab, or wasn't properly stopped.
>
> **Fix:**
> ```bash
> # Find and kill the process using port 8000
> lsof -ti:8000 | xargs kill -9 2>/dev/null
>
> # If lsof is not available, use fuser instead
> fuser -k 8000/tcp 2>/dev/null
>
> # Then retry
> adk web
> ```

### 6. Cloud Shell session disconnected

> **Symptom:** Cloud Shell terminal is unresponsive, shows "Reconnecting...", or you return to a fresh terminal prompt without your virtual environment active.
>
> **Cause:** Cloud Shell timed out due to inactivity (default: 20 minutes idle) or a network interruption.
>
> **Fix:** See the [Environment Recovery](#environment-recovery) section for full recovery steps.

### 7. Using project name instead of project ID

> **Symptom:** `gcloud config set project` fails with `WARNING: You do not appear to have access to project [genai-workshop]` or similar.
>
> **Cause:** You used the **project name** (e.g., `genai-workshop`) instead of the **project ID** (e.g., `genai-workshop-123456`). These are different — the project ID is globally unique and often has a numeric suffix.
>
> **Fix:**
> 1. Find your project ID at [console.cloud.google.com/projectcreate](https://console.cloud.google.com/cloud-resource-manager) or from the project creation step.
> 2. The project ID appears below the project name field during creation. It may have a numeric suffix (e.g., `genai-workshop-449215`).
> ```bash
> gcloud config set project YOUR-ACTUAL-PROJECT-ID
> ```

---

## Step-by-Step Troubleshooting

### Step 3: Configure Google Cloud Services

#### Issue: Cannot create a Google Cloud project

> **Symptom:** The "Create" button is grayed out, or you get an error saying you've exceeded the project quota.
>
> **Cause:** Free-tier Google accounts have a limit on the number of projects. You may have hit that limit.
>
> **Fix:**
> 1. Go to [Resource Manager](https://console.cloud.google.com/cloud-resource-manager).
> 2. Delete any unused projects to free up quota.
> 3. Alternatively, reuse an existing project — just make sure it has billing enabled.

#### Issue: Cloud Shell authorization popup doesn't appear

> **Symptom:** Cloud Shell loads but you're never prompted to authorize, and `gcloud` commands fail with authentication errors.
>
> **Cause:** Pop-ups may be blocked by your browser, or you're using a browser extension that interferes with the authorization flow.
>
> **Fix:**
> 1. Ensure pop-ups are allowed for `shell.cloud.google.com`.
> 2. Try in an incognito window with extensions disabled.
> 3. If `gcloud` commands fail, manually re-authenticate:
> ```bash
> gcloud auth login
> ```

#### Issue: `gcloud config set project` shows a warning about access

> **Symptom:** `WARNING: You do not appear to have access to project [project-id], are you sure you wish to set property [core/project] to [project-id]?`
>
> **Cause:** The project ID is mistyped, or you're logged into Cloud Shell with a different Google account than the one that owns the project.
>
> **Fix:**
> ```bash
> # Check which account is active
> gcloud auth list
>
> # Verify the project ID exists and you have access
> gcloud projects describe YOUR-PROJECT-ID
> ```
> If the account is wrong, re-authenticate with the correct account:
> ```bash
> gcloud auth login
> ```

#### Issue: Enabling the Vertex AI API fails

> **Symptom:** `ERROR: (gcloud.services.enable) FAILED_PRECONDITION: Billing account for project is not found.`
>
> **Cause:** No billing account is linked to the project.
>
> **Fix:**
> 1. Go to [Billing](https://console.cloud.google.com/billing/linkedaccount).
> 2. Link the **Google Cloud Platform Trial Billing Account**.
> 3. Retry the API enablement:
> ```bash
> gcloud services enable aiplatform.googleapis.com
> ```

---

### Step 4: Create a Python Virtual Environment

#### Issue: `uv` command not found

> **Symptom:** `command not found: uv` when running `uv venv --python 3.12`.
>
> **Cause:** `uv` may not be pre-installed in your Cloud Shell environment, or your PATH doesn't include its install location.
>
> **Fix:**
> ```bash
> # Install uv
> curl -LsSf https://astral.sh/uv/install.sh | sh
>
> # Reload your shell profile
> source ~/.bashrc  # or source ~/.zshrc
>
> # Verify installation
> uv --version
> ```

#### Issue: Python 3.12 not available

> **Symptom:** `uv venv --python 3.12` fails with an error about Python 3.12 not being found or not being available.
>
> **Cause:** Cloud Shell may not have Python 3.12 installed by default on some VM images.
>
> **Fix:**
> ```bash
> # Let uv download and manage Python 3.12 automatically
> uv python install 3.12
>
> # Then retry
> uv venv --python 3.12
> source .venv/bin/activate
> ```

#### Issue: Virtual environment activation doesn't show the prefix

> **Symptom:** After running `source .venv/bin/activate`, the terminal prompt doesn't show `(ai-agents-adk)`.
>
> **Cause:** Some shell configurations or custom prompts suppress the virtual environment prefix display.
>
> **Fix:** The virtual environment may still be active even without the visual prefix. Verify by checking:
> ```bash
> which python
> ```
> If it shows a path containing `.venv/bin/python`, the virtual environment is active. If it points to a system Python path, re-run:
> ```bash
> cd ~/ai-agents-adk
> source .venv/bin/activate
> ```

#### Issue: `uv pip install google-adk` fails with network errors

> **Symptom:** `error: Failed to download` or connection timeout errors during package installation.
>
> **Cause:** Temporary network issues in Cloud Shell or PyPI availability problems.
>
> **Fix:**
> ```bash
> # Retry the installation
> uv pip install google-adk
>
> # If it keeps failing, try with a longer timeout
> uv pip install --timeout 120 google-adk
> ```

---

### Step 5: Create an Agent

#### Issue: `adk` command not found

> **Symptom:** `command not found: adk` when running `adk create personal_assistant`.
>
> **Cause:** The `google-adk` package was not installed, or the virtual environment is not activated.
>
> **Fix:**
> ```bash
> # Activate the virtual environment first
> cd ~/ai-agents-adk
> source .venv/bin/activate
>
> # Verify adk is installed
> uv pip list | grep google-adk
>
> # If not listed, install it
> uv pip install google-adk
> ```

#### Issue: `adk create` prompts show an incorrect or blank project ID

> **Symptom:** During `adk create personal_assistant`, the `Enter Google Cloud project ID [...]` prompt shows a blank or wrong project ID in the brackets.
>
> **Cause:** The `gcloud config set project` command from Step 3 wasn't run, or was run in a different terminal session.
>
> **Fix:** Press Enter to skip past the prompt, then manually enter the correct project ID. Or exit the creation process (Ctrl+C), set the project, and retry:
> ```bash
> gcloud config set project YOUR-PROJECT-ID
> adk create personal_assistant
> ```

#### Issue: `personal_assistant` directory already exists

> **Symptom:** `adk create personal_assistant` fails because the directory already exists from a previous attempt.
>
> **Cause:** A previous (possibly incomplete) run of `adk create` already created the directory.
>
> **Fix:**
> ```bash
> # Remove the previous attempt
> rm -rf ~/ai-agents-adk/personal_assistant
>
> # Retry
> adk create personal_assistant
> ```

#### Issue: Selected wrong model or backend during `adk create`

> **Symptom:** You accidentally chose the wrong model (option 2 instead of 1) or wrong backend (Google AI instead of Vertex AI) during the interactive prompts.
>
> **Cause:** Misread the options during the interactive `adk create` flow.
>
> **Fix:** Delete the created directory and start over:
> ```bash
> rm -rf ~/ai-agents-adk/personal_assistant
> adk create personal_assistant
> ```
> Select: model → **1** (gemini-2.5-flash), backend → **2** (Vertex AI), verify project ID, press Enter for region.

---

### Step 6: Explore Agent Codes

#### Issue: `.env` file is not visible in Cloud Shell Editor

> **Symptom:** The `personal_assistant` folder in the Cloud Shell Editor shows `agent.py` and `__init__.py` but no `.env` file.
>
> **Cause:** Files starting with `.` (dotfiles) are hidden by default in the Cloud Shell Editor.
>
> **Fix:**
> 1. In the Cloud Shell Editor menu bar, click **View**.
> 2. Select **Toggle Hidden Files**.
> 3. The `.env` file should now appear in the file tree.
>
> Alternatively, verify its existence from the terminal:
> ```bash
> cat ~/ai-agents-adk/personal_assistant/.env
> ```

#### Issue: Cloud Shell Editor doesn't show `ai-agents-adk` folder

> **Symptom:** After clicking **File > Open Folder**, you can't find the `ai-agents-adk` folder.
>
> **Cause:** The folder was created in a different directory than `$HOME`, or the editor opened to a different root path.
>
> **Fix:**
> 1. In the **Open Folder** dialog, navigate to `/home/YOUR_USERNAME/`.
> 2. Select the `ai-agents-adk` folder.
> 3. If the folder doesn't exist, verify from the terminal:
> ```bash
> ls ~/ai-agents-adk
> ```
> If missing, go back to Step 4 and recreate it.

#### Issue: Editor top menu bar is not visible

> **Symptom:** The Cloud Shell Editor loads but the top menu bar (File, Edit, View, etc.) is not visible.
>
> **Cause:** The Cloud Shell Editor window may be in a minimized state or the menu is collapsed.
>
> **Fix:** Click the folder icon (hamburger menu) on the left sidebar and choose **Open Folder** from there, as mentioned in the codelab. Alternatively, try resizing the browser window or pressing F11 to toggle fullscreen.

---

### Step 7: Run the Agent on the Terminal

#### Issue: `This API method requires billing to be enabled`

> **Symptom:** After running `adk run personal_assistant` and sending a message, you get `{'message': 'This API method requires billing to be enabled'}`.
>
> **Cause:** The project's billing account is not linked or has expired.
>
> **Fix:**
> 1. Verify the `.env` file has the correct project ID:
>    ```bash
>    cat ~/ai-agents-adk/personal_assistant/.env
>    ```
> 2. Go to the [linked billing account page](https://console.cloud.google.com/billing/linkedaccount).
> 3. Link **Google Cloud Platform Trial Billing Account** if not already linked.
> 4. Retry: type `exit` to stop the agent, then run `adk run personal_assistant` again.

#### Issue: `Vertex AI API has not been used in project`

> **Symptom:** Error message containing `Vertex AI API has not been used in project` followed by your project number.
>
> **Cause:** The Vertex AI API wasn't enabled in Step 3, or the enablement hasn't propagated yet.
>
> **Fix:**
> ```bash
> # Exit the agent first (type "exit" or press Ctrl+C)
> gcloud services enable aiplatform.googleapis.com
> ```
> Wait for `Operation/... finished successfully`, then retry `adk run personal_assistant`.

#### Issue: `ModuleNotFoundError: No module named 'google'`

> **Symptom:** Running `adk run personal_assistant` throws a Python import error for `google.adk` or related modules.
>
> **Cause:** The virtual environment is not activated, or `google-adk` was never installed.
>
> **Fix:**
> ```bash
> cd ~/ai-agents-adk
> source .venv/bin/activate
> uv pip install google-adk
> adk run personal_assistant
> ```

#### Issue: Agent starts but doesn't respond to messages

> **Symptom:** You see the `[user]:` prompt and type a message, but no response appears — the terminal hangs.
>
> **Cause:** Network connectivity issues between Cloud Shell and the Vertex AI endpoint, or the API request is being rate-limited.
>
> **Fix:**
> 1. Wait up to 30 seconds — the first request may take longer due to cold start.
> 2. If still no response, press Ctrl+C to exit and retry:
>    ```bash
>    adk run personal_assistant
>    ```
> 3. Verify your internet connectivity from Cloud Shell:
>    ```bash
>    curl -s -o /dev/null -w "%{http_code}" https://us-central1-aiplatform.googleapis.com
>    ```
>    If this doesn't return `404` (expected — it means the endpoint is reachable), check your network.

#### Issue: `adk run` runs from the wrong directory

> **Symptom:** `adk run personal_assistant` returns `Error: Agent 'personal_assistant' not found` or similar path-related error.
>
> **Cause:** You're running the command from a directory that doesn't contain the `personal_assistant/` subdirectory.
>
> **Fix:**
> ```bash
> cd ~/ai-agents-adk
> adk run personal_assistant
> ```

#### Issue: Warning messages appear before the `[user]:` prompt

> **Symptom:** Several warning lines (e.g., deprecation warnings, logging warnings) appear before the `[user]:` prompt.
>
> **Cause:** These are informational warnings from underlying libraries. They do not affect functionality.
>
> **Fix:** No action needed. As long as you see `[user]:` at the end, the agent is running correctly. You can safely ignore these warnings.

---

### Step 8: Run the Agent on the Development Web UI

#### Issue: Port 8000 already in use

> **Symptom:** `ERROR: [Errno 98] Address already in use` when running `adk web`.
>
> **Cause:** A previous `adk run` or `adk web` process is still running.
>
> **Fix:**
> ```bash
> # Kill any process using port 8000
> lsof -ti:8000 | xargs kill -9 2>/dev/null
>
> # If lsof is not available, use fuser instead
> fuser -k 8000/tcp 2>/dev/null
>
> # Retry
> adk web
> ```

#### Issue: Web Preview shows a blank page or "Unable to connect"

> **Symptom:** After clicking Web Preview and setting port 8000, the browser tab shows a blank page, "This site can't be reached", or "Unable to connect".
>
> **Cause:** The `adk web` server hasn't fully started, the port number is wrong, or the Cloud Shell preview proxy is having issues.
>
> **Fix:**
> 1. Verify `adk web` is running — check the terminal for `Uvicorn running on http://127.0.0.1:8000`.
> 2. Make sure you entered port **8000** (not 8080 or another port) in the Web Preview dialog.
> 3. Wait a few seconds and refresh the browser tab.
> 4. If still blank, try Ctrl+Click on the `http://localhost:8000` link directly from the terminal instead of using Web Preview.

#### Issue: Cannot find the Web Preview button

> **Symptom:** You can't locate the Web Preview button in Cloud Shell.
>
> **Cause:** The button location varies depending on the Cloud Shell UI version.
>
> **Fix:** Look for a square icon with an eye or a globe-like icon in the Cloud Shell toolbar (top right area of the terminal). It may also appear as an icon with a pop-out arrow. If you can't find it:
> 1. Try Ctrl+Click (or Cmd+Click on Mac) directly on the `http://localhost:8000` URL in the terminal output.
> 2. Alternatively, open a new browser tab and navigate to the Cloud Shell proxy URL manually.

#### Issue: `adk web` exits immediately after starting

> **Symptom:** The server starts but immediately stops, or you see an error about missing dependencies.
>
> **Cause:** The virtual environment is not activated, or required packages are missing.
>
> **Fix:**
> ```bash
> cd ~/ai-agents-adk
> source .venv/bin/activate
> uv pip install google-adk
> adk web
> ```

#### Issue: Chat messages in the Web UI return errors

> **Symptom:** The Web UI loads, but sending messages returns error responses or error banners in the chat interface.
>
> **Cause:** Same root causes as terminal errors — billing, API enablement, or project configuration issues in `.env`.
>
> **Fix:** Check the terminal where `adk web` is running for detailed error messages. Then refer to the [Step 7 troubleshooting](#step-7-run-the-agent-on-the-terminal) section for the specific error.

#### Issue: Forgot to stop `adk run` before running `adk web`

> **Symptom:** You try to run `adk web` but the terminal is still in the `adk run` interactive session showing `[user]:`.
>
> **Cause:** The `adk run` session is still active in the current terminal.
>
> **Fix:** Type `exit` at the `[user]:` prompt to close `adk run`, then run `adk web`:
> ```bash
> exit
> adk web
> ```
> Alternatively, press Ctrl+C to force-stop the `adk run` process.

---

## General FAQ

### Q: Which Google account should I use — personal Gmail or Workspace?

**A:** For self-study with trial credits, use your **personal Gmail** account. Workspace accounts may have organizational policies that restrict project creation or billing. If you're in an instructor-led workshop, follow the instructor's guidance.

### Q: Can I use Google AI (option 1) instead of Vertex AI (option 2) as the backend?

**A:** The codelab is designed for Vertex AI (option 2). If you chose Google AI (option 1), the `.env` file will have different variables and you'll need a Google AI API key instead of a Cloud project. Delete the `personal_assistant` directory and re-run `adk create personal_assistant`, selecting **Vertex AI** this time.

### Q: Do I need to keep the Cloud Shell tab open the entire time?

**A:** Yes, while the agent is running (`adk run` or `adk web`), the Cloud Shell terminal must remain open. If you close the tab or the session times out, the agent process stops. See [Environment Recovery](#environment-recovery) to get back on track.

### Q: The codelab mentions `personal-assistant` in some places and `personal_assistant` in others. Which is correct?

**A:** The directory and Python package name is **`personal_assistant`** (with an underscore). Python package names cannot contain hyphens. If you see `personal-assistant` referenced, it is the same agent — just use `personal_assistant` (underscore) in all commands and file paths.

### Q: Can I run this codelab locally instead of Cloud Shell?

**A:** Yes, but you'll need additional setup not covered in the codelab: install `gcloud` CLI, authenticate with `gcloud auth application-default login`, install Python 3.12, and install `uv`. The codelab steps assume Cloud Shell's pre-configured environment.

### Q: I see deprecation warnings — is that a problem?

**A:** No. Deprecation warnings from Python libraries do not affect the agent's functionality. They indicate that a future library version may change an API. You can safely ignore them for this codelab.

---

## Environment Recovery

Use this section if you've been disconnected from Cloud Shell, your session timed out, or you need to get back to a known-good state.

### Recovering from Cloud Shell Disconnection

If your Cloud Shell session was disconnected or timed out:

1. **Re-open Cloud Shell** at [shell.cloud.google.com](https://shell.cloud.google.com) and click **Authorize** if prompted.

2. **Verify your project is set correctly:**
   ```bash
   gcloud config get-value project
   ```
   If it returns the wrong project or blank:
   ```bash
   gcloud config set project YOUR-PROJECT-ID
   ```

3. **Navigate to your working directory:**
   ```bash
   cd ~/ai-agents-adk
   ```

4. **Re-activate the virtual environment:**
   ```bash
   source .venv/bin/activate
   ```
   You should see `(ai-agents-adk)` in your prompt.

5. **Verify your agent files are intact:**
   ```bash
   ls personal_assistant/
   ```
   You should see: `agent.py`, `__init__.py`, `.env`

6. **Verify the `.env` file has correct values:**
   ```bash
   cat personal_assistant/.env
   ```
   Confirm `GOOGLE_CLOUD_PROJECT` and `GOOGLE_CLOUD_LOCATION` are set correctly.

7. **Resume where you left off** — your files persist across Cloud Shell reconnections, so you can pick up from whatever step you were on.

### Verifying Your Environment State

Run these checks to confirm your environment is properly configured:

```bash
# Check project
echo "Project: $(gcloud config get-value project 2>/dev/null)"

# Check virtual environment
echo "Python: $(which python)"

# Check adk is available
echo "ADK: $(which adk 2>/dev/null || echo 'NOT FOUND - activate venv')"

# Check .env contents
echo "--- .env ---"
cat ~/ai-agents-adk/personal_assistant/.env 2>/dev/null || echo ".env NOT FOUND"

# Verify Vertex AI API is enabled
gcloud services list --enabled --filter="name:aiplatform.googleapis.com" --format="value(name)" 2>/dev/null || echo "Cannot check APIs"
```

Expected output:
- **Project:** Your Google Cloud project ID
- **Python:** A path containing `.venv/bin/python`
- **ADK:** A path containing `.venv/bin/adk`
- **.env:** Shows `GOOGLE_GENAI_USE_VERTEXAI=1`, your project ID, and `us-central1`
- **Vertex AI API:** Shows `aiplatform.googleapis.com`

### Starting Over from a Specific Step

If you need to restart from a particular step:

- **From Step 3 (Configure Google Cloud Services):** No cleanup needed. Just re-run `gcloud config set project` and the API enablement command.

- **From Step 4 (Create a Python Virtual Environment):**
  ```bash
  cd ~
  rm -rf ai-agents-adk
  mkdir ai-agents-adk && cd ai-agents-adk
  uv venv --python 3.12
  source .venv/bin/activate
  uv pip install google-adk
  ```

- **From Step 5 (Create an Agent):**
  ```bash
  cd ~/ai-agents-adk
  source .venv/bin/activate
  rm -rf personal_assistant
  adk create personal_assistant
  ```
  Then follow the interactive prompts (model: 1, backend: 2, verify project ID, Enter for region).

- **From Step 7 or 8 (Run the Agent):**
  ```bash
  cd ~/ai-agents-adk
  source .venv/bin/activate
  # Kill any lingering processes
  lsof -ti:8000 | xargs kill -9 2>/dev/null || fuser -k 8000/tcp 2>/dev/null
  # Run terminal mode
  adk run personal_assistant
  # Or run web UI mode
  adk web
  ```

- **Complete restart:**
  ```bash
  cd ~
  rm -rf ai-agents-adk
  ```
  Then start from Step 4 of the codelab.
