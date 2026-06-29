# ci-workflows

Shared reusable GitHub Actions workflows for Lennar engineering teams. The repo
is public so callers in any org can reference it without cross-org agreements. No
secrets or prompts live here; consuming repos supply those at call time.

## Workflows

| Workflow | What it does |
|---|---|
| [`codex-pr-review.yml`](.github/workflows/codex-pr-review.yml) | AI code review on each PR push using OpenAI Codex. Posts a structured review with inline diff comments. |

## codex-pr-review

### How it works

1. A caller workflow in a service repo triggers on `pull_request` (or
   `pull_request_target` for repos that accept fork PRs).
2. It invokes this workflow via
   `uses: modsy/ci-workflows/.github/workflows/codex-pr-review.yml@main`.
3. The workflow:
   - Checks out the PR merge commit (`refs/pull/N/merge`) with `persist-credentials: false`.
   - Fetches PR metadata, the diff, and any linked issues.
   - Optionally runs `pre_install_command` (e.g. `npm ci`) and reports the outcome in the prompt.
   - Reads the review prompt from `.github/codex/prompts/codex-pr-review.md` in the caller repo, or falls back to a built-in default.
   - Runs the Codex CLI (`@openai/codex`) in ephemeral sandbox mode.
   - Parses the JSON response between `BEGIN_REVIEW_JSON` / `END_REVIEW_JSON` markers.
   - Posts a PR review (verdict plus inline comments) via `github.rest.pulls.createReview`.

### Security model

| Concern | How it is addressed |
|---|---|
| Secrets isolation | `CODEX_ACCESS_TOKEN` / `CODEX_AUTH_JSON` are stored per org or repo and passed at call time. Never stored here. |
| Fork PR exfiltration | Callers using `pull_request_target` must include the fork guard from the example (`if:` on the job, not a separate prerequisite job). |
| Prompt privacy | The prompt lives in the caller repo and is read at runtime. |
| Credential leak via git | `persist-credentials: false` removes `GITHUB_TOKEN` from git config, so scripts cannot push or fetch with it. |
| Script injection | GitHub context values reach the shell as environment variables, never interpolated into `run:` scripts. |
| Sandbox | Codex runs with `--ephemeral --sandbox workspace-write`: ephemeral container, no persistent state, write access scoped to the workspace. |

### Inputs

| Input | Type | Required | Default | Description |
|---|---|---|---|---|
| `pr_number` | number | No | auto | PR to review. Auto-resolved from the event; pass explicitly for `workflow_dispatch`. |
| `codex_version` | string | No | `0.142.0` | Pinned `@openai/codex` version. Pin in the caller for reproducible reviews. |
| `pre_install_command` | string | No | `""` | Command run before review (e.g. `npm ci`). Runs with `continue-on-error`; the outcome is reported in the prompt. |

### Secrets

One of the following is required:

| Secret | Description |
|---|---|
| `CODEX_AUTH_JSON` | Full `~/.codex/auth.json` content. Preferred for ChatGPT Business/Enterprise accounts: it includes the refresh token, so Codex routes through `chatgpt.com`. |
| `CODEX_ACCESS_TOKEN` | Personal access token (`at-*`), OpenAI API key (`sk-*`), or ChatGPT OAuth JWT. |

### Outputs

| Output | Description |
|---|---|
| `review_url` | URL of the posted PR review. |

## Adoption

### 1. Add the secret

Add `CODEX_AUTH_JSON` (preferred) or `CODEX_ACCESS_TOKEN` at the repo level
(`Settings > Secrets and variables > Actions`) or org level for sharing. For Lennar
ChatGPT Business/Enterprise accounts, use `CODEX_AUTH_JSON`: the refresh token lets
Codex authenticate through `chatgpt.com` instead of `api.openai.com`, which needs
different scopes.

### 2. Add the caller workflow

Create `.github/workflows/pr-review.yml`:

```yaml
name: PR Review

on:
  pull_request:
    types: [opened, synchronize, reopened, ready_for_review]
  workflow_dispatch:
    inputs:
      pr_number:
        description: "PR number to review"
        required: true
        type: number

permissions:
  contents: read
  pull-requests: write
  issues: write

concurrency:
  group: codex-pr-review-${{ github.event.pull_request.number || inputs.pr_number }}
  cancel-in-progress: false

jobs:
  codex-review:
    if: >-
      github.event_name == 'workflow_dispatch' ||
      (github.event.pull_request.head.repo.full_name == github.repository &&
       github.event.pull_request.draft == false)
    uses: modsy/ci-workflows/.github/workflows/codex-pr-review.yml@main
    with:
      pr_number: ${{ github.event.pull_request.number || inputs.pr_number }}
      pre_install_command: "npm ci"   # "" for Python; add a build for monorepos
      codex_version: "0.142.0"
    secrets:
      CODEX_ACCESS_TOKEN: ${{ secrets.CODEX_ACCESS_TOKEN }}
      CODEX_AUTH_JSON: ${{ secrets.CODEX_AUTH_JSON }}
```

### 3. Add a prompt (recommended)

Create `.github/codex/prompts/codex-pr-review.md` with your stack-specific review
instructions. It must describe the stack, list the review dimensions, and instruct
Codex to emit JSON between `BEGIN_REVIEW_JSON` / `END_REVIEW_JSON`:

```json
{
  "verdict": "APPROVE | REQUEST_CHANGES | COMMENT",
  "summary": "One paragraph.",
  "comments": [{ "path": "src/file.ts", "line": 42, "body": "..." }]
}
```

Without a prompt file, a built-in default is used. See
[timmy's prompt](https://github.com/modsy/timmy/blob/main/.github/codex/prompts/codex-pr-review.md)
for reference.

## Versioning

Pin to a commit SHA for reproducibility, or `@main` for the latest:

```yaml
uses: modsy/ci-workflows/.github/workflows/codex-pr-review.yml@<sha>
```

## Troubleshooting

| Symptom | Cause / fix |
|---|---|
| Review does not appear | Check the `codex-review` job logs and the "Authenticate Codex" step. Confirm the secret is set and scoped to the repo. |
| 401 from `api.openai.com/v1/responses` | Token routes to the OpenAI API but lacks `api.responses.write`. Use `CODEX_AUTH_JSON` so Codex routes through `chatgpt.com`. |
| "invalid agent identity JWT format" | The `CODEX_ACCESS_TOKEN` value is not a valid JWT/PAT. Use `CODEX_AUTH_JSON` instead. |
| "Cannot resolve a PR number" | Pass `pr_number` when triggering via `workflow_dispatch`. |
| Fork PRs not reviewed | Intentional under `pull_request_target`. A maintainer can trigger a review via `workflow_dispatch` after inspecting the fork. |
| `pre_install_command` fails | Install is `continue-on-error`; the prompt reports it. Fix the command, or build workspaces for monorepos. |
| bubblewrap / sandbox errors | The workflow sets `kernel.unprivileged_userns_clone=1` for Linux runners, required by Codex's sandbox. |

## Contributing

Maintained by the modsy platform team. Open a PR for changes that benefit consuming
teams; note any breaking input or behavior change in the description.
