# ci-workflows

Shared reusable GitHub Actions workflows for Lennar engineering teams. The repo
is public so callers in any org can reference it without cross-org agreements. No
secrets live here (consuming repos pass those at call time); the review prompt **is**
bundled here and applied to every caller, so consuming repos carry no review logic.

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
   - Auto-detects the stack and installs dependencies (npm/yarn/pnpm, uv/pip, go) — zero config. `pre_install_command` is an optional override for non-standard layouts.
   - Reads the review prompt from **this shared repo** (`.github/codex/prompts/codex-pr-review.md`, version-locked to the workflow's own commit) — never from the PR under review. The prompt is stack-agnostic: it detects the stack and reads the repo's own `AGENTS.md` / `CLAUDE.md` for conventions. Consuming repos carry no prompt.
   - Runs the Codex CLI (`@openai/codex`) in ephemeral sandbox mode.
   - Parses the JSON response between `BEGIN_REVIEW_JSON` / `END_REVIEW_JSON` markers.
   - Posts a PR review (verdict plus inline comments) via `github.rest.pulls.createReview`.

### Security model

| Concern | How it is addressed |
|---|---|
| Secrets isolation | `CODEX_ACCESS_TOKEN` / `CODEX_AUTH_JSON` are stored per org or repo and passed at call time. Never stored here. |
| Fork PR exfiltration | Callers using `pull_request_target` must include the fork guard from the example (`if:` on the job, not a separate prerequisite job). |
| Prompt integrity | The prompt is bundled in this shared repo and read from the workflow's own commit, never from the PR checkout. A PR cannot alter what the credentialed reviewer runs. |
| Credential leak via git | `persist-credentials: false` removes `GITHUB_TOKEN` from git config, so scripts cannot push or fetch with it. |
| Script injection | GitHub context values reach the shell as environment variables, never interpolated into `run:` scripts. |
| Sandbox | Codex runs with `--ephemeral --sandbox workspace-write`: ephemeral container, no persistent state, write access scoped to the workspace. |

### Inputs

| Input | Type | Required | Default | Description |
|---|---|---|---|---|
| `pr_number` | number | No | auto | PR to review. Auto-resolved from the event; pass explicitly for `workflow_dispatch`. |
| `codex_version` | string | No | `0.142.0` | Pinned `@openai/codex` version. Pin in the caller for reproducible reviews. |
| `pre_install_command` | string | No | `""` | Optional override for dependency setup. Leave empty and the stack is auto-detected (npm/yarn/pnpm, uv/pip, go). Set only for non-standard layouts (e.g. a package in a monorepo subdir). |

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

permissions:
  contents: read
  pull-requests: write
  issues: write

concurrency:
  group: codex-pr-review-${{ github.event.pull_request.number }}
  cancel-in-progress: false

jobs:
  codex-review:
    # Same-repo guard: never run for fork PRs (keeps credentials away from
    # untrusted code). Do not add workflow_dispatch — an arbitrary PR number
    # would bypass this guard.
    if: >-
      github.event.pull_request.head.repo.full_name == github.repository &&
      github.event.pull_request.draft == false
    uses: modsy/ci-workflows/.github/workflows/codex-pr-review.yml@main
    with:
      pr_number: ${{ github.event.pull_request.number }}
      codex_version: "0.142.0"
    secrets:
      CODEX_ACCESS_TOKEN: ${{ secrets.CODEX_ACCESS_TOKEN }}
      CODEX_AUTH_JSON: ${{ secrets.CODEX_AUTH_JSON }}
```

### 3. That's it — no prompt to add

Do **not** add a prompt to your repo. The review prompt is bundled in this shared
repo ([`.github/codex/prompts/codex-pr-review.md`](.github/codex/prompts/codex-pr-review.md))
and applied to every caller. It is stack-agnostic: it detects your stack and reads your
repo's `AGENTS.md` / `CLAUDE.md` / `.agents/rules/` for project conventions. To change
review behavior for everyone, edit the prompt here; to tune it for one repo, document the
convention in that repo's `AGENTS.md`.

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
| Dependencies not installed | Auto-detect is best-effort and `continue-on-error`; the review still runs statically. For a nested monorepo package, set `pre_install_command` to your install command. |
| bubblewrap / sandbox errors | The workflow sets `kernel.unprivileged_userns_clone=1` for Linux runners, required by Codex's sandbox. |

## Contributing

Maintained by the modsy platform team. Open a PR for changes that benefit consuming
teams; note any breaking input or behavior change in the description.
