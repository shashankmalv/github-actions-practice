# GitHub Actions Practice Project

A hands-on project to learn and practice GitHub Actions workflows. Built with a simple Flask web app as the target application.

---

## Project Structure

```
├── app.py                  # Flask app (main entry point)
├── src/app.py              # Simple Python script
├── templates/index.html    # Frontend HTML page
├── test_app.py             # Pytest tests for the Flask app
├── requirements.txt        # Python dependencies
├── Dockerfile              # Docker image definition
├── docker-compose.yml      # Docker Compose for deployment
└── .github/
    ├── workflows/          # 29 workflow files (see below)
    └── actions/
        └── setup-and-greet/
            └── action.yml  # Custom composite action
```

---

## Application

A minimal Flask app with two routes:

- `GET /` — renders the HTML portfolio page
- `GET /health` — returns `"Server is up and running"`

Run locally:
```bash
pip install -r requirements.txt
python app.py
```

Run tests:
```bash
pytest
```

---

## Workflows

### 1. `hello.yml` — Hello World
**Trigger:** `workflow_dispatch`

The starting point. Checks out code, prints a greeting, shows the current date, branch name, repo files, and the current user. Great for understanding the basics of jobs and steps.

---

### 2. `cahe.yml` — Python CI with Cache
**Trigger:** `workflow_dispatch`

Demonstrates pip dependency caching using `actions/cache@v4`. The cache key is based on the OS and a hash of `requirements.txt`, so the cache invalidates automatically when dependencies change. Speeds up repeated workflow runs significantly.

---

### 3. `python-ci.yml` — Python CI Pipeline
**Trigger:** `workflow_dispatch`

A full CI pipeline for the Flask app:
1. Checkout code
2. Setup Python 3.11
3. Install dependencies
4. Run `flake8` for linting
5. Run `bandit` for SAST (static security analysis)
6. Run `pytest` for tests

---

### 4. `tests.yml` — Run Tests
**Trigger:** `push` (all branches)

Runs on every push. Installs dependencies and runs `pytest`. This workflow's name (`Run Tests`) is referenced by `deploy-after-tests.yml` via `workflow_run`.

---

### 5. `deploy-after-tests.yml` — Deploy After Tests
**Trigger:** `workflow_run` (when `Run Tests` completes)

Shows how to chain workflows. Waits for the `Run Tests` workflow to finish, checks its conclusion, and only deploys if tests passed. Uses `github.event.workflow_run.conclusion` to make the decision.

---

### 6. `docker-publish.yml` — Docker Build & Push
**Trigger:** `workflow_dispatch`

Builds the Docker image and pushes it to Docker Hub with two tags:
- `latest`
- `sha-<short-commit-sha>` (e.g., `sha-a1b2c3d`)

Uses `docker/login-action` and `docker/build-push-action`. Requires `DOCKER_USERNAME` (variable) and `DOCKER_TOKEN` (secret) to be configured in the repo settings.

---

### 7. `artifacts.yml` — Artifact Upload
**Trigger:** `workflow_dispatch`

Generates a test report file and uploads it as a GitHub Actions artifact using `actions/upload-artifact@v4`. The artifact can be downloaded from the workflow run page in GitHub.

---

### 8. `artifact-transfer.yml` — Artifact Transfer Between Jobs
**Trigger:** `workflow_dispatch`

Shows how to pass files between jobs (which run on separate runners) using artifacts:
- `job-upload` creates a file and uploads it
- `job-download` downloads it and reads the content

Uses `needs:` to enforce job ordering.

---

### 9. `multi-job.yml` — Multi-Job Pipeline
**Trigger:** `workflow_dispatch`

A simple sequential pipeline: `build` → `test` → `deploy`. Each job uses `needs:` to depend on the previous one. Demonstrates how jobs run on separate runners and how to control execution order.

---

### 10. `job-outputs.yaml` — Job Outputs
**Trigger:** `workflow_dispatch`

Shows how to pass data between jobs using outputs:
- `generate-date` job captures today's date and sets it as an output via `$GITHUB_OUTPUT`
- `print-date` job reads it using `${{ needs.generate-date.outputs.today }}`

---

### 11. `env-vars.yml` — Environment Variables
**Trigger:** `workflow_dispatch`

Demonstrates the three scopes of environment variables:
- **Workflow-level** `env:` — available to all jobs
- **Job-level** `env:` — available to all steps in that job
- **Step-level** `env:` — available only in that step

Also shows built-in variables like `GITHUB_SHA` and `GITHUB_ACTOR`.

---

### 12. `conditionals.yml` — Conditional Steps & Jobs
**Trigger:** `workflow_dispatch`

Covers conditional execution using `if:`:
- Run a step only on the `main` branch
- Run a step only when the previous step failed (`failure()`)
- Use `continue-on-error: true` to let a failing step not block the workflow
- Run an entire job only on `push` events (`github.event_name == 'push'`)

---

### 13. `matrix.yml` — Matrix Strategy
**Trigger:** `workflow_dispatch`

Runs the same job across multiple Python versions (`3.11`, `3.12`) in parallel using `strategy.matrix`. `fail-fast: true` means if one version fails, the others are cancelled immediately.

---

### 14. `multi-os.yml` — Multi-OS Runners
**Trigger:** `workflow_dispatch`

Runs jobs on three different runner types simultaneously:
- `ubuntu-latest`
- `windows-latest`
- `macos-latest`

Shows how to print OS info and that the same workflow can target different operating systems.

---

### 15. `manual.yml.yml` — Manual Workflow with Inputs
**Trigger:** `workflow_dispatch` with inputs

Demonstrates `workflow_dispatch` inputs. When triggered manually from the GitHub UI, the user can select an environment (`staging` or `production`). The value is accessed via `github.event.inputs.environment`.

---

### 16. `schedule.yml` — Scheduled Workflow (Daily)
**Trigger:** `schedule` (cron: `0 0 * * *`) + `workflow_dispatch`

Runs automatically every day at midnight UTC using a cron expression. Also supports manual triggering for testing. Good for nightly builds, cleanup jobs, or health checks.

---

### 17. `scheduled-tasks.yml` — Scheduled Tasks with Health Check
**Trigger:** `workflow_dispatch` (schedule commented out)

A more practical scheduled workflow that performs an HTTP health check against a URL using `curl`. Fails the workflow if the HTTP status code is not 200. The schedule (Monday 2:30 AM + every 6 hours) is commented out for practice.

---

### 18. `smart-triggers.yml` — Path-Based Triggers
**Trigger:** `push` to `main` or `release/*` — only when `src/**` or `app/**` changes

Shows how to use `paths:` to avoid running workflows on irrelevant changes. The workflow only triggers when actual code files change, not docs or config.

---

### 19. `ignore-docs.yml` — Ignore Docs Changes
**Trigger:** `push` to `main` or `release/*` — ignores `*.md` and `docs/**`

The opposite of `smart-triggers.yml`. Uses `paths-ignore:` to skip the workflow when only documentation files are changed. Saves CI minutes on doc-only commits.

---

### 20. `pr-checks.yml` — PR Validation Checks
**Trigger:** `pull_request` targeting `main`

Runs three parallel validation jobs on every PR:
- **File size check** — fails if any file is larger than 1MB
- **Branch name check** — enforces naming convention (`feature/`, `fix/`, `docs/`)
- **PR body check** — warns if the PR description is empty

---

### 21. `pr-lifecycle.yml` — PR Lifecycle Events
**Trigger:** `pull_request` on `opened`, `synchronize`, `reopened`, `closed`

Demonstrates how to react to different PR events. Prints PR title, author, source/target branches, and runs a special step only when the PR is actually merged (not just closed).

---

### 22. `reusable-build.yml` — Reusable Workflow (called)
**Trigger:** `workflow_call`

A reusable workflow that can be called by other workflows. Accepts:
- **Inputs:** `app_name`, `environment`
- **Secrets:** `docker_token`
- **Outputs:** `build_version` (a generated version string)

This is the workflow that gets called by `call-build.yml`.

---

### 23. `call-build.yml` — Call Reusable Workflow (caller)
**Trigger:** `workflow_dispatch`

Calls `reusable-build.yml` using `uses: ./.github/workflows/reusable-build.yml` and passes inputs and secrets to it. After the build job finishes, a second job prints the `build_version` output returned by the reusable workflow.

---

### 24. `external-trigger.yml` — External / Repository Dispatch
**Trigger:** `repository_dispatch` with type `deploy-request`

Shows how to trigger a workflow from an external system (e.g., another service, a script, or a webhook) using the GitHub API. The external caller can pass a custom payload, and the workflow reads it via `github.event.client_payload`.

Trigger it with:
```bash
curl -X POST \
  -H "Authorization: token <YOUR_TOKEN>" \
  -H "Accept: application/vnd.github.v3+json" \
  https://api.github.com/repos/<owner>/<repo>/dispatches \
  -d '{"event_type":"deploy-request","client_payload":{"environment":"production"}}'
```

---

### 25. `secret-read.yml` — Reading Secrets
**Trigger:** `workflow_dispatch`

Shows how to access repository secrets using `${{ secrets.MY_SECRET_NAME }}`. Note: GitHub automatically masks secret values in logs, so you won't see the actual value printed — just `***`.

---

### 26. `preinstalled-tools.yml` — Pre-installed Tools
**Trigger:** `workflow_dispatch`

Checks which tools come pre-installed on the `ubuntu-latest` runner: Docker, Python 3, Node.js, and Git. Useful for understanding what's available without needing a setup step.

---

### 27. `smart-pipeline.yml` — Smart Pipeline with Branch Logic
**Trigger:** `workflow_dispatch`

Runs `lint` and `test` jobs in parallel, then a `summary` job that depends on both. The summary job prints the commit message and detects whether the push was to `main` or a feature branch using `github.ref`.

---

### 28. `self-hosted.yml` — Self-Hosted Runner
**Trigger:** `workflow_dispatch`

Demonstrates how to target a self-hosted runner using labels (`[self-hosted, my-linux-runner]`). Requires a self-hosted runner to be registered in the repo settings. Will fail if no matching runner is available.

---

### 29. `use-composite.yml` — Custom Composite Action
**Trigger:** `workflow_dispatch`

Uses the custom composite action defined in `.github/actions/setup-and-greet/action.yml`. Passes `name: "Shashank"` and `language: "hi"` as inputs, and reads the `greeted` output after the action runs.

---

## Custom Action: `setup-and-greet`

Located at `.github/actions/setup-and-greet/action.yml`.

A composite action (reusable set of steps packaged as a single action) that:
1. Prints a greeting in English or Hindi based on the `language` input
2. Prints the current date and runner OS
3. Sets `greeted=true` as an output via `$GITHUB_OUTPUT`

**Inputs:**
| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `name` | yes | — | Name to greet |
| `language` | no | `en` | `en` for English, `hi` for Hindi |

**Outputs:**
| Output | Description |
|--------|-------------|
| `greeted` | `true` if greeting was executed |

---

## Secrets & Variables Required

| Name | Type | Used In | Description |
|------|------|---------|-------------|
| `DOCKER_TOKEN` | Secret | `docker-publish.yml`, `call-build.yml` | Docker Hub access token |
| `MY_SECRET_MESSAGE` | Secret | `secret-read.yml` | Any secret value for practice |
| `DOCKER_USERNAME` | Variable | `docker-publish.yml` | Docker Hub username |
| `DOCKERHUB_USER` | Variable | `docker-compose.yml` | Docker Hub username for compose |

---

## Concepts Covered

| Concept | Workflow(s) |
|---------|-------------|
| Basic jobs & steps | `hello.yml` |
| Dependency caching | `cahe.yml` |
| CI pipeline (lint, test, security) | `python-ci.yml` |
| Artifacts (upload/download) | `artifacts.yml`, `artifact-transfer.yml` |
| Job dependencies (`needs`) | `multi-job.yml`, `artifact-transfer.yml` |
| Job outputs | `job-outputs.yaml` |
| Environment variables (3 scopes) | `env-vars.yml` |
| Conditional steps & jobs | `conditionals.yml` |
| Matrix builds | `matrix.yml` |
| Multi-OS runners | `multi-os.yml` |
| Manual inputs (`workflow_dispatch`) | `manual.yml.yml` |
| Scheduled runs (cron) | `schedule.yml`, `scheduled-tasks.yml` |
| Path-based triggers | `smart-triggers.yml`, `ignore-docs.yml` |
| PR events & validation | `pr-checks.yml`, `pr-lifecycle.yml` |
| Reusable workflows | `reusable-build.yml`, `call-build.yml` |
| External triggers (repository_dispatch) | `external-trigger.yml` |
| Secrets | `secret-read.yml` |
| Pre-installed tools | `preinstalled-tools.yml` |
| Workflow chaining (`workflow_run`) | `tests.yml`, `deploy-after-tests.yml` |
| Docker build & push | `docker-publish.yml` |
| Self-hosted runners | `self-hosted.yml` |
| Composite custom actions | `use-composite.yml` |
| Branch-aware logic | `smart-pipeline.yml` |
