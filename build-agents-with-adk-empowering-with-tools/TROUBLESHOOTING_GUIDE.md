# Troubleshooting Guide: Building AI Agents with ADK: Empowering with Tools

> **Codelab URL:** https://codelabs.developers.google.com/devsite/codelabs/build-agents-with-adk-empowering-with-tools
>
> **Last updated:** 2026-02-24

This guide helps you diagnose and fix common issues encountered while following the **Building AI Agents with ADK: Empowering with Tools** codelab. If you're stuck, start with the [Quick Fixes](#quick-fixes) section — it covers the most frequent problems.

---

## Table of Contents

- [Quick Fixes](#quick-fixes)
- [Step-by-Step Troubleshooting](#step-by-step-troubleshooting)
  - [Step 2: Before You Begin](#step-2-before-you-begin)
  - [Step 3: Getting Started — Your Base Agent](#step-3-getting-started--your-base-agent)
  - [Step 4: Build a Custom Tool for Currency Exchange](#step-4-build-a-custom-tool-for-currency-exchange)
  - [Step 5: Integrate with Built-in Google Search Tool](#step-5-integrate-with-built-in-google-search-tool)
  - [Step 6: Leverage LangChain's Wikipedia Tool](#step-6-leverage-langchains-wikipedia-tool)
  - [Step 7: Clean Up](#step-7-clean-up)
- [General FAQ](#general-faq)
- [Environment Recovery](#environment-recovery)

---

## Quick Fixes

The most common issues across all steps. **Check these first** before diving into step-specific troubleshooting.

### 1. Virtual environment not activated

> **Symptom:** Running `adk web` returns `command not found: adk` or `ModuleNotFoundError: No module named 'google.adk'`.
>
> **Cause:** The Python virtual environment was deactivated — either because you opened a new terminal tab, Cloud Shell reconnected, or you never activated it.
>
> **Fix:**
> ```bash
> cd ~/ai-agents-adk
> source .venv/bin/activate
> ```
> You should see `(ai-agents-adk)` prefixing your terminal prompt.

### 2. Port already in use when running `adk web`

> **Symptom:** `ERROR: [Errno 98] Address already in use` or `ERROR: Could not bind to port 8000` when running `adk web`.
>
> **Cause:** A previous `adk web` process wasn't stopped with Ctrl+C before you started a new one. This codelab requires you to stop and restart `adk web` multiple times (Steps 4, 5, and 6), and forgetting to stop it is the most common mistake.
>
> **Fix:**
> ```bash
> # Find and kill the process using port 8000
> lsof -ti:8000 | xargs kill -9 2>/dev/null || fuser -k 8000/tcp 2>/dev/null
>
> # Then retry
> adk web
> ```

### 3. Wrong working directory

> **Symptom:** `touch personal_assistant/custom_functions.py` returns `No such file or directory`, or `adk web` returns `Error: Agent 'personal_assistant' not found`.
>
> **Cause:** You're not in the `~/ai-agents-adk` directory. Every command in this codelab must run from there.
>
> **Fix:**
> ```bash
> cd ~/ai-agents-adk
> pwd
> # Should output: /home/YOUR_USERNAME/ai-agents-adk
> ```

### 4. File created in wrong location

> **Symptom:** After creating `custom_functions.py`, `custom_agents.py`, or `third_party_tools.py`, the agent throws `ModuleNotFoundError` or `ImportError` when running `adk web`.
>
> **Cause:** The file was created in the `ai-agents-adk/` root directory instead of inside the `personal_assistant/` subdirectory.
>
> **Fix:**
> ```bash
> # Check where the files are
> ls ~/ai-agents-adk/personal_assistant/
>
> # If a file is in the wrong place, move it
> mv ~/ai-agents-adk/custom_functions.py ~/ai-agents-adk/personal_assistant/
> mv ~/ai-agents-adk/custom_agents.py ~/ai-agents-adk/personal_assistant/
> mv ~/ai-agents-adk/third_party_tools.py ~/ai-agents-adk/personal_assistant/
> ```

### 5. `agent.py` not fully replaced

> **Symptom:** `ImportError` or `NameError` when running `adk web` — e.g., `cannot import name 'AgentTool'` or `name 'google_search_agent' is not defined`.
>
> **Cause:** The codelab instructs you to **replace the entire contents** of `agent.py` at each step. If you appended code instead of replacing, or only partially replaced, old imports and definitions conflict with new ones.
>
> **Fix:** Open `agent.py` in the Cloud Shell Editor, select all (Ctrl+A), delete, and paste the exact code block from the current step. The file should contain only the code shown in that step — nothing from previous steps.

### 6. Cloud Shell session disconnected

> **Symptom:** Cloud Shell terminal is unresponsive, shows "Reconnecting...", or you return to a fresh terminal prompt without your virtual environment active.
>
> **Cause:** Cloud Shell timed out due to inactivity (default: 20 minutes idle) or a network interruption.
>
> **Fix:** See the [Environment Recovery](#environment-recovery) section for full recovery steps.

### 7. `requests` module not found in `custom_functions.py`

> **Symptom:** `ModuleNotFoundError: No module named 'requests'` when the agent tries to use the `get_fx_rate` tool.
>
> **Cause:** The `requests` library is typically installed as a dependency of `google-adk`, but in some environments it may not be present.
>
> **Fix:**
> ```bash
> cd ~/ai-agents-adk
> source .venv/bin/activate
> uv pip install requests
> ```

### 8. Forgot to stop `adk web` before modifying code

> **Symptom:** After editing `agent.py` or creating new files, the agent doesn't reflect your changes — it still uses the old tool configuration.
>
> **Cause:** The `adk web` server loaded your agent code at startup. Code changes require a restart.
>
> **Fix:** Go to the terminal running `adk web`, press **Ctrl+C** to stop it, then run `adk web` again:
> ```bash
> # In the terminal where adk web is running:
> # Press Ctrl+C first, then:
> adk web
> ```

---

## Step-by-Step Troubleshooting

### Step 2: Before You Begin

#### Issue: Trial credit redemption link doesn't work

> **Symptom:** Navigating to the trial redemption portal URL returns an error, "no credits available", or a blank page.
>
> **Cause:** The trial credit link may be expired (these are typically tied to specific events or time windows), or you're not using an incognito window as instructed.
>
> **Fix:**
> 1. Open a new **incognito/private** browser window.
> 2. Log in with your **personal Gmail** account (not a Workspace account).
> 3. If the link is expired, use an existing Google Cloud project with billing enabled, or start a [free trial](https://cloud.google.com/free).

#### Issue: Used Workspace account instead of personal Gmail

> **Symptom:** The redemption portal blocks you, or your project has organizational policy restrictions that prevent enabling APIs or creating resources.
>
> **Cause:** Google Workspace accounts are managed by your organization and may have restrictions on billing, API enablement, or project creation.
>
> **Fix:** Open an incognito window and use a **personal Gmail account** for the trial. If you must use a Workspace account, contact your organization's admin to ensure Vertex AI API access is permitted.

---

### Step 3: Getting Started — Your Base Agent

#### Issue: Don't have the base agent from the Foundation codelab

> **Symptom:** You're starting this codelab directly (Path B) and need to set up the base agent, but the linked steps in the Foundation codelab are confusing to follow out of context.
>
> **Cause:** Path B requires completing 4 specific steps from the Foundation codelab. Jumping between codelabs can be disorienting.
>
> **Fix:** Follow these steps in order from the Foundation codelab:
> 1. [Configure Google Cloud Services](https://codelabs.developers.google.com/devsite/codelabs/build-agents-with-adk-foundation#2) — set project ID and enable Vertex AI API
> 2. [Create a Python virtual environment](https://codelabs.developers.google.com/devsite/codelabs/build-agents-with-adk-foundation#3) — create `ai-agents-adk` directory, set up venv, install `google-adk`
> 3. [Create an agent](https://codelabs.developers.google.com/devsite/codelabs/build-agents-with-adk-foundation#4) — run `adk create personal_assistant`
> 4. [Run the agent on the Development UI](https://codelabs.developers.google.com/devsite/codelabs/build-agents-with-adk-foundation#7) — verify `adk web` works
>
> If you have issues with any of these steps, refer to the [Foundation codelab troubleshooting guide](../build-agents-with-adk-foundation/TROUBLESHOOTING_GUIDE.md).

#### Issue: Missing files from the Foundation codelab

> **Symptom:** The `personal_assistant/` directory is missing `agent.py`, `__init__.py`, or `.env`.
>
> **Cause:** The Foundation codelab was not completed, or files were accidentally deleted.
>
> **Fix:** Check what's present:
> ```bash
> ls -la ~/ai-agents-adk/personal_assistant/
> ```
> You should see at least: `__init__.py`, `agent.py`, `.env`. If any are missing, re-run:
> ```bash
> cd ~/ai-agents-adk
> source .venv/bin/activate
> rm -rf personal_assistant
> adk create personal_assistant
> ```
> Then follow the `adk create` prompts: model → **1** (gemini-2.5-flash), backend → **2** (Vertex AI), verify project ID, press Enter for region.

#### Issue: Virtual environment from Foundation codelab is gone

> **Symptom:** `source .venv/bin/activate` returns `No such file or directory`.
>
> **Cause:** The `.venv` directory was deleted, or the `ai-agents-adk` directory was recreated without the virtual environment.
>
> **Fix:**
> ```bash
> cd ~/ai-agents-adk
> uv venv --python 3.12
> source .venv/bin/activate
> uv pip install google-adk
> ```

---

### Step 4: Build a Custom Tool for Currency Exchange

#### Issue: `touch personal_assistant/custom_functions.py` — No such file or directory

> **Symptom:** `touch: cannot touch 'personal_assistant/custom_functions.py': No such file or directory`
>
> **Cause:** You're not in the `ai-agents-adk` directory, or the `personal_assistant` directory doesn't exist.
>
> **Fix:**
> ```bash
> cd ~/ai-agents-adk
> ls personal_assistant/
> # If the directory doesn't exist, go back to Step 3 (Path B)
> ```

#### Issue: Python indentation errors in `custom_functions.py`

> **Symptom:** `IndentationError: unexpected indent` or `IndentationError: expected an indented block` when running the agent.
>
> **Cause:** Copy-pasting from the codelab web page sometimes introduces inconsistent whitespace (mixing tabs and spaces), or extra/missing indentation.
>
> **Fix:** Open `custom_functions.py` in the Cloud Shell Editor and verify:
> - The `def get_fx_rate` function body uses consistent indentation (either all tabs or all spaces — the codelab uses 8-space indentation in the docstring and function body).
> - The docstring (`"""..."""`) is indented inside the function.
> - The `base_url`, `api_url`, `response`, and `return` lines are all at the same indentation level.
>
> If unsure, delete the file contents and re-paste from the codelab, or type the code manually.

#### Issue: `get_fx_rate` returns `None` or the agent says it couldn't get the rate

> **Symptom:** The agent uses the `get_fx_rate` tool but reports it couldn't fetch the exchange rate, or the response is empty.
>
> **Cause:** The external API (`hexarate.paikama.co`) may be down, rate-limited, or inaccessible from Cloud Shell's network. The function returns `None` when the response status code isn't 200.
>
> **Fix:**
> 1. Test the API directly from Cloud Shell:
>    ```bash
>    curl -s "https://hexarate.paikama.co/api/rates/latest/SGD?target=JPY"
>    ```
> 2. If this returns an error or empty response, the API is temporarily unavailable. Wait a few minutes and retry.
> 3. If the API is consistently down, the tool won't work but the rest of the codelab can still proceed — the Google Search agent (Step 5) can answer exchange rate questions as a fallback.

#### Issue: Agent doesn't use the `get_fx_rate` tool

> **Symptom:** When asking "What is the exchange rate from SGD to JPY?", the agent answers from its training data instead of calling the `get_fx_rate` function.
>
> **Cause:** The `agent.py` file wasn't updated correctly — the `tools` list may not include `FunctionTool(get_fx_rate)`, or the import is missing.
>
> **Fix:** Verify `agent.py` contains exactly:
> ```python
> from google.adk.agents import Agent
> from google.adk.tools import FunctionTool
>
> from .custom_functions import get_fx_rate
>
> root_agent = Agent(
>     model='gemini-2.5-flash',
>     name='root_agent',
>     description='A helpful assistant for user questions.',
>     instruction='Answer user questions to the best of your knowledge',
>     tools=[FunctionTool(get_fx_rate)]
> )
> ```
> Then restart `adk web` (Ctrl+C, then `adk web`).

#### Issue: `ImportError: cannot import name 'get_fx_rate' from '.custom_functions'`

> **Symptom:** Running `adk web` fails immediately with an import error referencing `custom_functions`.
>
> **Cause:** The `custom_functions.py` file is either empty, in the wrong directory, has a syntax error, or was named differently (e.g., `custom_function.py` without the 's').
>
> **Fix:**
> ```bash
> # Verify the file exists in the right location
> ls ~/ai-agents-adk/personal_assistant/custom_functions.py
>
> # Check it has content
> wc -l ~/ai-agents-adk/personal_assistant/custom_functions.py
>
> # Verify the function name matches
> grep "def get_fx_rate" ~/ai-agents-adk/personal_assistant/custom_functions.py
> ```
> If the file is missing or empty, recreate it with the code from the codelab.

---

### Step 5: Integrate with Built-in Google Search Tool

#### Issue: `ImportError: cannot import name 'google_search' from 'google.adk.tools'`

> **Symptom:** Running `adk web` fails with an import error for `google_search`.
>
> **Cause:** The installed version of `google-adk` may not include the `google_search` built-in tool, or the import path has changed in a newer version.
>
> **Fix:**
> ```bash
> cd ~/ai-agents-adk
> source .venv/bin/activate
>
> # Check the installed version
> uv pip show google-adk
>
> # Update to the latest version
> uv pip install --upgrade google-adk
> ```
> If the error persists after upgrading, check the [ADK documentation](https://google.github.io/adk-docs/tools/) for the current import path for the Google Search tool.

#### Issue: `ImportError: cannot import name 'AgentTool' from 'google.adk.tools.agent_tool'`

> **Symptom:** Running `adk web` fails with an import error for `AgentTool`.
>
> **Cause:** The installed version of `google-adk` may use a different module path for `AgentTool`.
>
> **Fix:**
> ```bash
> # Update google-adk
> uv pip install --upgrade google-adk
>
> # If still failing, check if AgentTool moved
> python -c "from google.adk.tools.agent_tool import AgentTool; print('OK')"
> ```
> If the import path has changed, check the [ADK tools documentation](https://google.github.io/adk-docs/tools/) for the current location.

#### Issue: Google Search tool returns permission errors or empty results

> **Symptom:** The agent delegates to `google_search_agent`, but the response contains a permission error or returns no results.
>
> **Cause:** The Google Search grounding feature requires the Vertex AI API to be enabled (which you did in the Foundation codelab setup). However, there may be additional API enablement needed, or your project may not have access to grounding features.
>
> **Fix:**
> 1. Verify the Vertex AI API is enabled:
>    ```bash
>    gcloud services list --enabled --filter="name:aiplatform.googleapis.com"
>    ```
> 2. If the issue is about grounding/search specifically, check the terminal running `adk web` for detailed error messages — they usually indicate which API or permission is missing.
> 3. If you see quota or billing errors, verify your billing account is active at [Billing](https://console.cloud.google.com/billing/linkedaccount).

#### Issue: `custom_agents.py` named incorrectly

> **Symptom:** `ImportError: No module named '.custom_agents'` or `cannot import name 'google_search_agent'`.
>
> **Cause:** The file was named `custom_agent.py` (singular) instead of `custom_agents.py` (plural), or is in the wrong directory.
>
> **Fix:**
> ```bash
> # Check the exact filename
> ls ~/ai-agents-adk/personal_assistant/custom_agent*
>
> # If it's singular, rename it
> mv ~/ai-agents-adk/personal_assistant/custom_agent.py ~/ai-agents-adk/personal_assistant/custom_agents.py
> ```

#### Issue: Agent uses wrong tool for the question

> **Symptom:** When asking about weather, the agent uses `get_fx_rate` instead of `google_search_agent`, or vice versa.
>
> **Cause:** The agent's LLM decides which tool to use based on descriptions. This is non-deterministic and may occasionally route incorrectly.
>
> **Fix:** This is expected behavior — LLM-based tool selection is probabilistic. Try rephrasing the question to be more explicit:
> - For exchange rates: "Use the exchange rate tool to check SGD to JPY rate"
> - For weather: "Search for the current weather forecast in Tokyo"
>
> If tool routing is consistently wrong, verify that `custom_agents.py` has a descriptive `description` field for the `google_search_agent` and that `custom_functions.py` has a clear docstring for `get_fx_rate`.

---

### Step 6: Leverage LangChain's Wikipedia Tool

#### Issue: `uv pip install langchain-community wikipedia` fails

> **Symptom:** Installation errors, dependency conflicts, or network timeout when installing `langchain-community` and `wikipedia`.
>
> **Cause:** Package version conflicts between `langchain-community` and existing packages in the virtual environment, or network issues.
>
> **Fix:**
> ```bash
> cd ~/ai-agents-adk
> source .venv/bin/activate
>
> # Retry with verbose output to see what's failing
> uv pip install langchain-community wikipedia -v
>
> # If there are version conflicts, try installing with relaxed constraints
> uv pip install --upgrade langchain-community wikipedia
> ```

#### Issue: Forgot to stop `adk web` before installing packages

> **Symptom:** You ran `uv pip install langchain-community wikipedia` while `adk web` was still running, and now the agent behaves unpredictably.
>
> **Cause:** Installing packages while the server is running can lead to inconsistent module state. The running process loaded the old package versions.
>
> **Fix:**
> 1. Stop `adk web` with **Ctrl+C** in the terminal where it's running.
> 2. Verify the packages installed correctly:
>    ```bash
>    uv pip list | grep -E "langchain-community|wikipedia"
>    ```
> 3. Restart the agent:
>    ```bash
>    adk web
>    ```

#### Issue: `ImportError: cannot import name 'LangchainTool' from 'google.adk.tools.langchain_tool'`

> **Symptom:** Running `adk web` fails with an import error for `LangchainTool`.
>
> **Cause:** The `google-adk` version may not include LangChain integration, or `langchain-community` is not installed.
>
> **Fix:**
> ```bash
> # Verify both packages are installed
> uv pip list | grep -E "google-adk|langchain"
>
> # Update google-adk to ensure LangChain support
> uv pip install --upgrade google-adk
>
> # Verify the import works
> python -c "from google.adk.tools.langchain_tool import LangchainTool; print('OK')"
> ```

#### Issue: `ImportError: No module named 'langchain_community'`

> **Symptom:** `adk web` starts but fails when trying to import `langchain_community` from `third_party_tools.py`.
>
> **Cause:** The `langchain-community` package was not installed, or was installed in a different Python environment.
>
> **Fix:**
> ```bash
> cd ~/ai-agents-adk
> source .venv/bin/activate
> uv pip install langchain-community wikipedia
>
> # Verify installation
> python -c "from langchain_community.tools import WikipediaQueryRun; print('OK')"
> ```

#### Issue: Wikipedia tool returns empty or error responses

> **Symptom:** The agent delegates to the Wikipedia tool, but the response is empty or shows "No good Wikipedia Search Result was found".
>
> **Cause:** The Wikipedia API query didn't match any articles, or the API is temporarily unavailable.
>
> **Fix:**
> 1. Test the Wikipedia tool directly:
>    ```bash
>    python -c "
>    from langchain_community.tools import WikipediaQueryRun
>    from langchain_community.utilities import WikipediaAPIWrapper
>    tool = WikipediaQueryRun(api_wrapper=WikipediaAPIWrapper(top_k_results=1, doc_content_chars_max=3000))
>    print(tool.run('Kyoto'))
>    "
>    ```
> 2. If the direct test works but the agent doesn't use it correctly, verify the `description` in `third_party_tools.py` matches the codelab (it should mention "historical and cultural information").
> 3. If the direct test also fails, there may be a network issue from Cloud Shell to the Wikipedia API. Wait and retry.

#### Issue: `third_party_tools.py` file naming or location error

> **Symptom:** `ImportError: No module named '.third_party_tools'` when running `adk web`.
>
> **Cause:** The file is misnamed (e.g., `thirdparty_tools.py`, `third_party_tool.py`) or is in the wrong directory.
>
> **Fix:**
> ```bash
> # Check the exact filename and location
> ls ~/ai-agents-adk/personal_assistant/third_party*
>
> # The file must be named exactly: third_party_tools.py
> # And located at: ~/ai-agents-adk/personal_assistant/third_party_tools.py
> ```

#### Issue: Agent has too many tools and routes to the wrong one

> **Symptom:** At this point the agent has three tools (`get_fx_rate`, `google_search_agent`, and `langchain_wikipedia_tool`). When asking historical questions, it uses Google Search instead of Wikipedia, or vice versa.
>
> **Cause:** The LLM's tool selection is based on descriptions and is non-deterministic. With three tools available, routing can occasionally be suboptimal.
>
> **Fix:** This is expected behavior. The tool descriptions guide the LLM's selection, but it's not guaranteed. To encourage correct routing:
> - For historical/cultural queries, phrase them as: "Tell me about the history of..."
> - For real-time information (weather, news), phrase them as: "What is the current..."
> - For exchange rates, phrase them as: "What is the exchange rate from X to Y?"
>
> Check the Events tab in the `adk web` UI to see which tool was called and verify the routing.

---

### Step 7: Clean Up

#### Issue: `gcloud services disable aiplatform.googleapis.com` fails

> **Symptom:** Error message about dependent services or resources still using the API.
>
> **Cause:** Other services or resources in the project depend on the Vertex AI API.
>
> **Fix:**
> ```bash
> # Force disable (only if you're sure no other services depend on it)
> gcloud services disable aiplatform.googleapis.com --force
> ```
> Alternatively, skip disabling the API if you plan to continue with the next codelab in the series.

---

## General FAQ

### Q: Do I need to complete the Foundation codelab first?

**A:** It's strongly recommended. This codelab (Part 2) builds directly on the agent created in Part 1 ([Building AI Agents with ADK: The Foundation](https://codelabs.developers.google.com/devsite/codelabs/build-agents-with-adk-foundation)). If you choose Path B (starting fresh), you must complete 4 setup steps from the Foundation codelab. The Foundation codelab's troubleshooting guide can help with issues in those steps.

### Q: Why do I keep having to stop and restart `adk web`?

**A:** The `adk web` dev server loads your agent code at startup. When you modify `agent.py`, create new files, or install new packages, the running server doesn't automatically pick up the changes. You must stop it (Ctrl+C) and restart it (`adk web`) for changes to take effect.

### Q: Can I use a different model instead of `gemini-2.5-flash`?

**A:** The codelab uses `gemini-2.5-flash` for both the `root_agent` and `google_search_agent`. You can try other Gemini models (e.g., `gemini-2.5-pro`), but be aware that different models may route tools differently and response quality/speed will vary. Stick with `gemini-2.5-flash` if you want results matching the codelab screenshots.

### Q: The agent sometimes gives different answers than the screenshots — is that normal?

**A:** Yes. LLM responses are non-deterministic. The exact wording, formatting, and even tool routing may differ between runs. What matters is that the agent uses the correct tool (check the Events tab) and returns relevant information.

### Q: My `agent.py` keeps getting overwritten. Should I save previous versions?

**A:** The codelab intentionally replaces `agent.py` at each step to progressively add tools. You don't need to save previous versions — each step's `agent.py` is self-contained and includes all tools from prior steps. The final version (from Step 6) is the most complete.

### Q: Do I need to keep the Cloud Shell tab open the entire time?

**A:** Yes, while the agent is running (`adk web`), the Cloud Shell terminal must remain open. If you close the tab or the session times out, the agent process stops. See [Environment Recovery](#environment-recovery) to get back on track.

### Q: The `hexarate.paikama.co` API is down. Can I still continue?

**A:** Yes. The `get_fx_rate` tool won't work, but you can still proceed with Steps 5 and 6 (Google Search and Wikipedia tools). The Google Search agent can also answer exchange rate questions using web search as a fallback.

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
   Depending on which step you were on, you should see some or all of: `__init__.py`, `agent.py`, `.env`, `custom_functions.py`, `custom_agents.py`, `third_party_tools.py`

6. **Verify the `.env` file has correct values:**
   ```bash
   cat personal_assistant/.env
   ```
   Confirm `GOOGLE_CLOUD_PROJECT` and `GOOGLE_CLOUD_LOCATION` are set correctly.

7. **Kill any stale processes** that may conflict:
   ```bash
   lsof -ti:8000 | xargs kill -9 2>/dev/null || fuser -k 8000/tcp 2>/dev/null
   ```

8. **Resume where you left off** — your files persist across Cloud Shell reconnections. Restart the agent:
   ```bash
   adk web
   ```

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

# Verify installed packages for this codelab
echo "--- Packages ---"
uv pip list 2>/dev/null | grep -E "google-adk|langchain-community|wikipedia|requests" || echo "Cannot list packages - activate venv first"

# Check folder structure
echo "--- Agent Files ---"
ls ~/ai-agents-adk/personal_assistant/ 2>/dev/null || echo "personal_assistant/ NOT FOUND"

# Verify Vertex AI API is enabled
gcloud services list --enabled --filter="name:aiplatform.googleapis.com" --format="value(name)" 2>/dev/null || echo "Cannot check APIs"
```

Expected output:
- **Project:** Your Google Cloud project ID
- **Python:** A path containing `.venv/bin/python`
- **ADK:** A path containing `.venv/bin/adk`
- **.env:** Shows `GOOGLE_GENAI_USE_VERTEXAI=1`, your project ID, and `us-central1`
- **Packages:** Shows `google-adk`, and (if you've reached Step 6) `langchain-community` and `wikipedia`
- **Agent Files:** Shows `__init__.py`, `agent.py`, `.env`, and any additional files created so far
- **Vertex AI API:** Shows `aiplatform.googleapis.com`

### Starting Over from a Specific Step

If you need to restart from a particular step:

- **From Step 3 (Getting Started):** Follow Path B — complete the 4 Foundation codelab steps linked in the codelab.

- **From Step 4 (Custom Tool for Currency Exchange):**
  ```bash
  cd ~/ai-agents-adk
  source .venv/bin/activate
  # Remove extra files created in later steps (if any)
  rm -f personal_assistant/custom_functions.py
  rm -f personal_assistant/custom_agents.py
  rm -f personal_assistant/third_party_tools.py
  # Kill any running adk processes
  lsof -ti:8000 | xargs kill -9 2>/dev/null || fuser -k 8000/tcp 2>/dev/null
  ```
  Then follow Step 4 from the beginning.

- **From Step 5 (Google Search Tool):**
  ```bash
  cd ~/ai-agents-adk
  source .venv/bin/activate
  rm -f personal_assistant/custom_agents.py
  rm -f personal_assistant/third_party_tools.py
  lsof -ti:8000 | xargs kill -9 2>/dev/null || fuser -k 8000/tcp 2>/dev/null
  ```
  Ensure `custom_functions.py` exists with the `get_fx_rate` function, then follow Step 5 from the beginning.

- **From Step 6 (LangChain Wikipedia Tool):**
  ```bash
  cd ~/ai-agents-adk
  source .venv/bin/activate
  rm -f personal_assistant/third_party_tools.py
  uv pip install langchain-community wikipedia
  lsof -ti:8000 | xargs kill -9 2>/dev/null || fuser -k 8000/tcp 2>/dev/null
  ```
  Ensure `custom_functions.py` and `custom_agents.py` exist with correct content, then follow Step 6 from the beginning.

- **Complete restart:**
  ```bash
  cd ~
  rm -rf ai-agents-adk
  ```
  Then start from the Foundation codelab (Path B in Step 3).
