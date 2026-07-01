# testmu-a2a-chat

This repository contains GitHub Actions workflows for testing chat agents using the [TestMu A2A CLI](https://testmu.ai).

---

## Repository Structure

```
testmu-a2a-chat/
├── .github/workflows/
│   ├── view-chat-results.yml        # Workflow 1: View results of a past run
│   └── run-chat-scenarios.yml       # Workflow 2: Generate and run chat scenarios
└── testmu/
    ├── config/
    │   └── testmu-a2a.yaml    # Agent config, scenario settings, evaluation thresholds
    ├── scenarios/
    │   ├── premera-chat-scenarios.csv   # Master scenario list (edit enabled column here)
    │   └── active-scenarios.csv         # Auto-generated at runtime (do not edit)
    ├── specs/                 # Spec files for scenario generation guidance
    └── reports/               # JUnit XML results written here at runtime
```

---

## Prerequisites

Add these secrets to your GitHub repository (**Settings → Secrets and variables → Actions**):

| Secret | Description |
|---|---|
| `TESTMU_USERNAME` | Your TestMu / LambdaTest username |
| `TESTMU_ACCESS_KEY` | Your TestMu / LambdaTest access key |

---

## Workflow 1 — View Chat Agent Results (`view-chat-results.yml`)

Fetches and displays the results of a **previously completed** run from the TestMu platform.

### How to run

1. Go to **Actions → View Chat Results → Run workflow**
2. Click **Run workflow**

### What it does

1. Installs `testmu-a2a-cli`
2. Calls `testmu-a2a results` with a hardcoded project ID and run ID
3. Outputs results in JSON format to `results.txt`

### Known limitations

- The project ID (`2a54a65a-...`) and run ID (`fbab002e-...`) are hardcoded in the workflow
- Update these values in [`.github/workflows/view-chat-results.yml`](.github/workflows/view-chat-results.yml) to fetch a different run

### Known errors (all CLI bugs — report to TestMu support)

**1. `testmu-a2a results` crashes with AttributeError**

```
AttributeError: 'str' object has no attribute 'get'
```

Occurs at `workflow_runner.py:244` inside `classify_results`. The API returns each result as a JSON string instead of a parsed object, so calling `.get("metrics", {})` fails. This affects any call to `testmu-a2a results`.

**2. `testmu-a2a call-results get <id>` returns blank table**

```
Call Result - BLANK
┏━━━━━━━━━━━━━━━━━━━━━━━━┳━━━━━━━━━━━━━━┓
┃ Metric                 ┃    Value     ┃
┡━━━━━━━━━━━━━━━━━━━━━━━━╇━━━━━━━━━━━━━━┩
└────────────────────────┴──────────────┘
```

The command runs without error but returns no metric data. Likely the result hasn't been processed yet or the call ID has no associated evaluation data.

**3. `testmu-a2a call-results get --audio <id>` returns server error**

```
Error: Failed: Server error. Try again later.
```

The audio retrieval endpoint returns a 500. This may be a backend issue on the TestMu platform side.

---

## Workflow 2 — Create and Run Chat Agent Scenarios (`run-chat-scenarios.yml`)

Generates new AI-driven chat test scenarios from your scenario CSV and runs them against your agent.

### How to run

1. Go to **Actions → Run Chat Scenario → Run workflow**
2. Optionally fill in:
   - **category** — restrict to a single category (e.g. `conversational-flow`, `intent-recognition`, `context-memory`, `error-handling`). Leave blank to run all.
   - **endpoint_profile** — a TestMu endpoint profile ID to override the agent URL in the YAML
3. Click **Run workflow**

### What it does step by step

| Step | Description |
|---|---|
| **Filter enabled scenarios** | Reads `premera-chat-scenarios.csv`, keeps only rows where `enabled = true`, writes them to `active-scenarios.csv` |
| **Patch CLI project ID parsing** | Fixes a bug in `testmu-a2a-cli` where the API returns `project_id` but the CLI looks for `id` |
| **Run chat scenario** | Runs `testmu-a2a run` using `testmu-a2a.yaml` as config |
| **Publish results** | Renders the JUnit XML report as a GitHub check with pass/fail annotations |
| **Upload results artifact** | Saves `results.xml` as a downloadable artifact from the Actions run page |

### How to control which scenarios run

Open `testmu/scenarios/premera-chat-scenarios.csv` and set the `enabled` column:

| `enabled` value | Result |
|---|---|
| `true` | scenario is included in the run |
| `false` | scenario is skipped |
| blank / null | scenario is skipped |

Commit the updated CSV and re-trigger the workflow.

### Important: how scenario generation works

- **Each run creates a brand new project** on the TestMu platform — there is no way to reuse a previous project or its scenarios
- **Pre-built scenarios cannot be run directly** — the CSV is uploaded as a spec document and the TestMu API AI-generates fresh conversation flows from it each time
- The `count: 30` in `testmu-a2a.yaml` controls the maximum number of scenarios generated per run
- Scenario categories are restricted to `conversational-flow`, `intent-recognition`, `context-memory`, and `error-handling` — security scenarios are disabled because they require a different evaluation endpoint (HyperExecute) not supported by this workflow

### Agent configuration

Edit `testmu/config/testmu-a2a.yaml` to change the agent under test:

```yaml
agent:
  endpoint: "https://api.vapi.ai/chat"   # replace with your agent URL
  method: POST
  headers:
    Content-Type: "application/json"
  body_template:
    message: "{{message}}"               # {{message}} is replaced with each test message
  response_path: "data.reply"            # JSONPath to extract the agent reply
```

### Known CLI bugs (patched in workflow)

The `testmu-a2a-cli` v0.1.1 has the following bugs. One is patched inline during the workflow run, the other requires TestMu support.

---

**Bug 1 — Project creation fails with "no ID in response" (patched)**

**Context:** Setting up a config file and running chat scenarios via CLI.

The CLI POSTs to `https://agent-testing.lambdatest.com/api/projects` to create a fresh project on every run. The API returns HTTP 200 with this response:

```json
{
  "success": true,
  "message": "Project created successfully",
  "data": {
    "project_id": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
    ...
  }
}
```

The CLI expects `data.id` but the API returns `data.project_id`, causing:

```
Error: Failed to create project - no ID in response.
```

**Workaround:** The "Patch CLI project ID parsing" step in `run-chat-scenarios.yml` fixes this with `sed` at runtime before the CLI runs. No action needed — the patch is already applied.

**Note:** Each run always creates a brand new project — there is no way to reuse an existing project or its scenarios across runs.

---

**Bug 2 — Results parsing crashes with AttributeError (not patched)**

The `results` command crashes with `AttributeError: 'str' object has no attribute 'get'`. See the [view-chat-results.yml known errors](#known-errors-all-cli-bugs--report-to-testmu-support) section above. Report to TestMu/LambdaTest support.

---

## Evaluation thresholds

Configured in `testmu/testmu-a2a.yaml`:

| Metric | Minimum passing score |
|---|---|
| Accuracy | 0.80 |
| Relevance | 0.80 |
| Coherence | 0.80 |
| Context retention | 0.75 |
