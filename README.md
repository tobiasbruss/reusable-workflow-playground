# reusable-workflow-playground

![Repository Discovery](https://github.com/YOUR_ORG/reusable-workflow-playground/actions/workflows/discover.yml/badge.svg)
![Orchestrate Repository Dispatches](https://github.com/YOUR_ORG/reusable-workflow-playground/actions/workflows/orchestrate.yml/badge.svg)

A GitHub Actions orchestration system that automatically discovers repositories using a shared reusable workflow and triggers GitHub Advanced Security (GHAS) findings collection across all of them at scale — using a GitHub App for authentication and cross-repository dispatch.

---

## Table of Contents

1. [How It Works](#how-it-works)
2. [GitHub App Setup](#github-app-setup)
3. [Repository Secrets](#repository-secrets)
4. [Target Repository Requirements](#target-repository-requirements)
5. [Usage](#usage)
6. [Workflow Reference](#workflow-reference)
7. [Troubleshooting](#troubleshooting)
8. [Contributing / Development](#contributing--development)

---

## How It Works

The system is composed of three workflow components that work together:

### Component 1 — `repo-task-core.yml` (Reusable Core)

The reusable workflow that performs the actual work. Target repositories call it via `uses:` in their own caller workflow. When invoked, it:

- Checks out the target repository
- Fetches open CodeQL, Dependabot, and Secret Scanning alerts via the GitHub API
- Uploads a `ghas-findings-<repo>` artifact containing all three alert JSON files
- Reports a `status`, `repo`, and `tokens_used` output back to the caller

Because it is called via `workflow_call`, `github.token` is automatically scoped to the **target repository** — no explicit token passing is required.

### Component 2 — `discover.yml` (Discovery)

Runs on a schedule (daily at 02:00 UTC) or manually. It:

1. Authenticates as the GitHub App using `APP_ID` + `APP_PRIVATE_KEY`
2. Lists all App installations via `GET /app/installations`
3. For each installation, enumerates accessible repositories
4. Scans each repo's `.github/workflows/` directory for any YAML file that references `repo-task-core.yml` via `uses:`
5. Outputs an `eligible-repos.json` artifact and a job output matrix

### Component 3 — `orchestrate.yml` (Orchestration)

Triggered automatically after a successful discovery run, or manually with an optional repo list override. It:

1. Resolves the target repo list (from the discovery artifact or a manual `repo_list` input)
2. Runs a matrix job — one runner per target repo — that:
   - Generates a fresh per-repo installation token scoped to the target
   - Triggers `workflow_dispatch` on the target repo's caller workflow
   - Records the HTTP result (triggered / skipped / failed / dry_run)
3. Writes a Markdown summary table to the Actions UI

### Data Flow

```mermaid
flowchart TD
    A[Schedule / workflow_dispatch] --> B

    subgraph orchestrator [reusable-workflow-playground]
        B[discover.yml\ndiscover job]
        B --> B1[Generate App JWT]
        B1 --> B2[List all App installations\nGET /app/installations]
        B2 --> B3[For each installation:\nlist repos]
        B3 --> B4[For each repo:\ncheck .github/workflows/ for\nuses: repo-task-core.yml]
        B4 --> B5[Build eligible-repos.json]
        B5 --> B6[Upload artifact\neligible-repos.json]
        B6 --> C

        C[orchestrate.yml\nprepare job]
        C --> C1[Download eligible-repos.json\nor parse repo_list input]
        C1 --> C2[Emit matrix output]
        C2 --> D

        D[orchestrate.yml\ndispatch job - matrix]
        D --> D1[Generate installation token\nscoped to target repo]
        D1 --> D2[POST workflow_dispatch\nto target repo workflow]
        D2 --> D3[Record success/failure]
        D3 --> E

        E[orchestrate.yml\nreport job]
        E --> E1[Aggregate results]
        E1 --> E2[Write GITHUB_STEP_SUMMARY]
    end

    subgraph target [Target Repository]
        F[caller-workflow.yml\nworkflow_dispatch trigger]
        F --> G[repo-task-core.yml\nworkflow_call]
        G --> G1[Fetch CodeQL alerts]
        G --> G2[Fetch Dependabot alerts]
        G --> G3[Fetch Secret Scanning alerts]
        G1 & G2 & G3 --> G4[Upload ghas-findings artifact]
    end

    D2 -->|triggers| F
```

---

## GitHub App Setup

The system uses a GitHub App for authentication. This allows it to generate short-lived, scoped installation tokens for each target repository without storing long-lived personal access tokens.

### Step 1 — Create the GitHub App

1. Go to **GitHub Settings → Developer Settings → GitHub Apps → New GitHub App**
   - For an organization app: `https://github.com/organizations/YOUR_ORG/settings/apps/new`
   - For a personal app: `https://github.com/settings/apps/new`

2. Fill in the app details:
   - **GitHub App name:** `repo-task-orchestrator` (or any name you prefer)
   - **Homepage URL:** URL of this repository (e.g. `https://github.com/YOUR_ORG/reusable-workflow-playground`)
   - **Webhook:** Uncheck "Active" — webhooks are not needed

### Step 2 — Set Required Permissions

Under **Repository permissions**, configure:

| Permission | Level | Why |
|------------|-------|-----|
| **Contents** | Read | Read `.github/workflows/` files during discovery |
| **Actions** | Read & Write | Trigger `workflow_dispatch` on target repositories |
| **Metadata** | Read | Required by GitHub for all Apps |

No organization permissions are required.

### Step 3 — Install the App

After creating the App:

1. Go to the App's settings page → **Install App**
2. Install it on every **target repository** that should be discovered and dispatched to
3. Alternatively, install it on your entire **organization** to cover all current and future repos

> **Note:** Only repositories where the App is installed will appear in discovery results. Repos without the App installed are silently skipped.

### Step 4 — Get the App ID

On the App's settings page, note the **App ID** (a numeric value, e.g. `123456`). You will need this for the `APP_ID` secret.

### Step 5 — Generate and Download the Private Key

1. On the App's settings page, scroll to **Private keys**
2. Click **Generate a private key**
3. A `.pem` file will be downloaded — keep it safe
4. The full contents of this file (including the `-----BEGIN RSA PRIVATE KEY-----` and `-----END RSA PRIVATE KEY-----` lines) become the `APP_PRIVATE_KEY` secret

> **Security:** Never commit the `.pem` file to any repository. Delete it from your local machine after storing it as a secret.

---

## Repository Secrets

Add the following secrets to **this repository** (`reusable-workflow-playground`) under **Settings → Secrets and variables → Actions → New repository secret**:

| Secret | Description |
|--------|-------------|
| `APP_ID` | The numeric GitHub App ID found on the App settings page (e.g. `123456`) |
| `APP_PRIVATE_KEY` | The full PEM content of the downloaded private key file, including the `-----BEGIN RSA PRIVATE KEY-----` header and footer lines |

Both secrets are used by `discover.yml` and `orchestrate.yml` to authenticate as the GitHub App and generate scoped installation tokens.

---

## Target Repository Requirements

For a repository to be discovered and dispatched to, it must have a caller workflow that meets the following requirements.

### Required Workflow Structure

The caller workflow must:

1. Be located in `.github/workflows/` (any filename ending in `.yml` or `.yaml`)
2. Declare `on: workflow_dispatch` with `dry_run` (boolean) and `runner` (string) inputs
3. Reference `repo-task-core.yml` via `uses:` pointing to this repository
4. Include `permissions: contents: read` and `security-events: read`

### Minimal Example Caller Workflow

```yaml
# .github/workflows/ghas-scan.yml
# Place this file in each target repository.

name: GHAS Security Scan

on:
  workflow_dispatch:
    inputs:
      dry_run:
        description: 'Simulate run without side-effects'
        type: boolean
        required: false
        default: false
      runner:
        description: 'Runner label to use'
        type: string
        required: false
        default: 'ubuntu-latest'

permissions:
  contents: read
  security-events: read

jobs:
  task:
    uses: YOUR_ORG/reusable-workflow-playground/.github/workflows/repo-task-core.yml@main
    with:
      dry_run: ${{ inputs.dry_run && 'true' || 'false' }}
      runner: ${{ inputs.runner || 'ubuntu-latest' }}
```

Replace `YOUR_ORG` with the GitHub organization or username that owns this (`reusable-workflow-playground`) repository.

> **Important:** The `dry_run` input must be converted from a boolean to a string (`'true'`/`'false'`) before passing it to `repo-task-core.yml`, which expects a `string` type input. The expression `${{ inputs.dry_run && 'true' || 'false' }}` handles this conversion.

---

## Usage

### Running Discovery Manually

1. Go to **Actions → Repository Discovery**
2. Click **Run workflow**
3. Click the green **Run workflow** button

Discovery will scan all App-installed repositories and upload an `eligible-repos.json` artifact. After it completes successfully, `orchestrate.yml` is triggered automatically.

---

### Running Orchestration Manually

1. Go to **Actions → Orchestrate Repository Dispatches**
2. Click **Run workflow**
3. Fill in the inputs:

| Input | Type | Default | Description |
|-------|------|---------|-------------|
| `repo_list` | string | _(empty)_ | JSON array of repo objects to target. If empty, uses the latest `eligible-repos` artifact from the most recent successful discovery run |
| `dry_run` | boolean | `false` | When `true`, logs dispatch intent but does not call the API — safe for testing |
| `runner` | string | `ubuntu-latest` | Runner label passed through to each target workflow's `runner` input |

#### Example `repo_list` input

To target specific repositories without running discovery first, provide a JSON array:

```json
[
  {"repo": "myorg/myrepo", "workflow_file": "ghas-scan.yml"},
  {"repo": "myorg/another-repo", "workflow_file": "security-scan.yml"}
]
```

If `workflow_file` is omitted or empty, the orchestrator will scan the target repo's `.github/workflows/` directory at dispatch time to find the matching caller workflow.

---

### Automatic Scheduling

The system runs automatically without manual intervention:

| Event | Time / Trigger |
|-------|----------------|
| **Discovery** runs | Daily at **02:00 UTC** |
| **Orchestration** runs | Automatically after each successful discovery run |

To change the discovery schedule, edit the `cron` expression in [`.github/workflows/discover.yml`](.github/workflows/discover.yml):

```yaml
schedule:
  - cron: '0 2 * * *'   # Daily at 02:00 UTC — adjust as needed
```

---

## Workflow Reference

| Workflow | File | Trigger | Purpose |
|----------|------|---------|---------|
| **Repo Task (Core)** | `.github/workflows/repo-task-core.yml` | `workflow_call` (called by target repos) | Fetches CodeQL, Dependabot, and Secret Scanning alerts; uploads `ghas-findings` artifact |
| **Repository Discovery** | `.github/workflows/discover.yml` | `schedule` (daily 02:00 UTC), `workflow_dispatch` | Discovers all App-installed repos that reference `repo-task-core.yml`; outputs `eligible-repos.json` |
| **Orchestrate Repository Dispatches** | `.github/workflows/orchestrate.yml` | `workflow_run` (after discovery), `workflow_dispatch` | Triggers `workflow_dispatch` on each eligible target repo; writes a dispatch report summary |

---

## Troubleshooting

### App not installed on a target repo

**Symptom:** A repository you expect to see is missing from discovery results.

**Fix:** Go to the GitHub App's settings page → **Install App** and install it on the missing repository (or on the entire organization).

---

### Target workflow missing `workflow_dispatch` trigger

**Symptom:** The dispatch job for a repo fails with HTTP status `422`.

**Fix:** Ensure the caller workflow in the target repository declares `on: workflow_dispatch` in its triggers. Workflows without this trigger cannot be dispatched remotely.

---

### `APP_PRIVATE_KEY` format issues

**Symptom:** Token generation fails with an authentication error.

**Fix:** The secret must contain the **full PEM file contents**, including the header and footer lines:

```
-----BEGIN RSA PRIVATE KEY-----
MIIEowIBAAKCAQEA...
...
-----END RSA PRIVATE KEY-----
```

GitHub Actions handles multi-line secrets correctly. Paste the entire file content as the secret value — do not strip the header/footer lines or add extra whitespace.

---

### Rate limiting for large numbers of repositories

**Symptom:** Some dispatch jobs fail intermittently, or discovery is slow.

**Context:** The GitHub REST API allows 5,000 authenticated requests per hour per token. The orchestration matrix is limited to `max-parallel: 5` concurrent dispatches by default to avoid secondary rate limits.

**Fix options:**
- For fewer than ~20 repos, you can safely increase `max-parallel` to `10` in [`orchestrate.yml`](.github/workflows/orchestrate.yml)
- For more than 256 repos, GitHub Actions matrix strategy has a hard limit of 256 entries — split the `repo_list` across multiple manual orchestration runs
- Discovery is API-intensive (one call per workflow file per repo) — if you have many repos, consider running discovery less frequently or filtering by known workflow filenames

---

### Orchestration not triggered after discovery

**Symptom:** Discovery completes successfully but orchestration does not start automatically.

**Context:** The `workflow_run` trigger only fires when the triggering workflow runs on the **default branch**. If `discover.yml` was triggered from a non-default branch, `orchestrate.yml` will not be triggered automatically.

**Fix:** Run orchestration manually via **Actions → Orchestrate Repository Dispatches → Run workflow**, or ensure discovery is always triggered from the default branch.

---

### GHAS alerts returning empty results

**Symptom:** The `ghas-findings` artifact is uploaded but all JSON files contain `[]`.

**Context:** `repo-task-core.yml` gracefully falls back to `[]` if GHAS APIs return errors (e.g. 404). This happens when:
- GitHub Advanced Security is not enabled on the target repository
- Dependabot alerts are not enabled
- The caller workflow's `permissions` block is missing `security-events: read`

**Fix:** Enable GHAS features on the target repository and ensure the caller workflow includes `permissions: security-events: read`.

---

## Contributing / Development

For a deep dive into the system design — including token scoping, data flow between jobs, rate limiting strategy, matrix output aggregation, and known edge cases — see [`docs/architecture.md`](docs/architecture.md).

The architecture document covers:
- GitHub App authentication flow (JWT → installation token)
- How data passes between `discover.yml` and `orchestrate.yml` via artifacts
- Matrix size limits and chunking strategies for large repo sets
- `dry_run` pass-through mechanics end-to-end
- GHAS permission requirements on target repositories
