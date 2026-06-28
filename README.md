# ci-workflows

Shared reusable GitHub Actions workflows for Lennar engineering teams.

This repo is **public** so workflows can be called from any org (`modsy`,
`product-org-len`, and future orgs) without cross-org enterprise agreements.
Secrets and prompts stay private in each service repo — nothing sensitive
lives here.

---

## Workflows

| Workflow | What it does |
|---|---|
| [`codex-pr-review.yml`](.github/workflows/codex-pr-review.yml) | AI code review on every PR push using OpenAI Codex. Posts a structured PR review with inline diff annotations, identical in behavior to GitHub Copilot code review. |

---

## `codex-pr-review` — AI PR Review

### How it works

1. A service repo's caller workflow triggers on `pull_request` (or
   `pull_request_target` for repos with fork PRs).
2. The caller invokes this shared workflow via
   `uses: modsy/ci-workflows/.github/workflows/codex-pr-review.yml@main`.
3. The shared workflow:
   - Checks out the PR's merge commit (`refs/pull/N/merge`) with
     `persist-credentials: false`.
   - Fetches PR metadata, the full diff, and any linked issues.
   - Optionally runs a `pre_install_command` (e.g. `npm ci`) and reports
     the outcome in the prompt.
   - Reads a custom review prompt from `.github/codex/prompts/codex-pr-review.md`
     in the caller repo (or falls back to a sensible built-in default).
   - Runs Codex CLI (`@openai/codex`) in ephemeral sandbox mode.
   - Parses the structured JSON response between `BEGIN_REVIEW_JSON` / `END_REVIEW_JSON` markers.
   - Posts the result as a PR review (verdict + inline diff comments) via
     `github.rest.pulls.createReview` — the same API used by GitHub Copilot.

### Security model

| Concern | How it is addressed |
|---|---|
| Secrets isolation | `CODEX_ACCESS_TOKEN` / `CODEX_AUTH_JSON` are stored in each org or repo and passed explicitly at call time. Never stored here. |
| Fork PR exfiltration | Callers using `pull_request_target` MUST include the fork guard shown in the examples (`if:` on the job; never on a separate prerequisite job). |
| Prompt privacy | The review prompt lives in each caller repo at `.github/codex/prompts/codex-pr-review.md` and is read at runtime. |
| Credential leak via git | `persist-credentials: false` removes `GITHUB_TOKEN` from the git config so scripts cannot use it to push or fetch. |
| Script injection | All GitHub context values are passed to shell via environment variables, never interpolated directly into `run:` scripts. |
| Sandbox | Codex runs with `--ephemeral --sandbox workspace-write` — ephemeral container, no persistent state, write access scoped to the workspace only. |

### Inputs

| Input | Type | Required | Default | Description |
|---|---|---|---|---|
| `pr_number` | number | No | auto | PR number to review. Auto-resolved from the event; pass explicitly for `workflow_dispatch`. |
| `codex_version` | string | No | `0.142.0` | Pinned version of `@openai/codex` to install. Pin this in your caller for reproducible reviews. |
| `pre_install_command` | string | No | `""` | Shell command to run before the review (e.g. `npm ci && npm run build --workspaces --if-present`). Runs with `continue-on-error`; outcome is reported in the prompt. |

### Secrets

| Secret | Required | Description |
|---|---|---|
| `CODEX_ACCESS_TOKEN` | One of these is required | Personal access token (`at-*`), OpenAI API key (`sk-*`), or ChatGPT OAuth JWT. |
| `CODEX_AUTH_JSON` | One of these is required | Full `auth.json` content from `~/.codex/auth.json`. Preferred over `CODEX_ACCESS_TOKEN` for ChatGPT Business/Enterprise accounts because it includes the refresh token, allowing Codex to route through `chatgpt.com`. |

Obtain `CODEX_AUTH_JSON` from your local machine after authenticating Codex:

```bash
cat ~/.codex/auth.json
```

Store the entire JSON blob as the `CODEX_AUTH_JSON` secret.

### Outputs

| Output | Description |
|---|---|
| `review_url` | URL of the posted PR review. |

---

## Adoption guide

### Step 1 — Add the secret

Add **`CODEX_AUTH_JSON`** (preferred) or **`CODEX_ACCESS_TOKEN`** as a secret in your repo:

- **Repo-level**: `Repo > Settings > Secrets and variables > Actions > New repository secret`
- **Org-level** (shared): `GitHub org > Settings > Secrets and variables > Actions > New org secret`

For ChatGPT Business or Enterprise accounts (Lennar), use `CODEX_AUTH_JSON` with the
full `~/.codex/auth.json` content. This gives Codex a refresh token so it can
stay authenticated through chatgpt.com rather than hitting `api.openai.com/v1/responses`
directly (which requires different scopes).

### Step 2 — Add the caller workflow

Create `.github/workflows/pr-review.yml` in your service repo:

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
      pre_install_command: "npm ci"   # adjust for your stack
      codex_version: "0.142.0"

    secrets:
      CODEX_ACCESS_TOKEN: ${{ secrets.CODEX_ACCESS_TOKEN }}
      CODEX_AUTH_JSON: ${{ secrets.CODEX_AUTH_JSON }}
```

For Python services, set `pre_install_command: ""` (no install needed before Codex runs).

For Node/frontend monorepos with workspace packages, build them too:

```yaml
pre_install_command: "npm ci && npm run build --workspaces --if-present"
```

### Step 3 — Add a custom prompt (recommended)

Create `.github/codex/prompts/codex-pr-review.md` in your repo. This is your
private, stack-specific review instruction set read at runtime.

The prompt must:
1. Describe the stack (frameworks, testing libraries, conventions).
2. List the review dimensions (correctness, security, performance, etc.).
3. Instruct Codex to output JSON between `BEGIN_REVIEW_JSON` and `END_REVIEW_JSON` markers
   with this shape:

```json
{
  "verdict": "APPROVE" | "REQUEST_CHANGES" | "COMMENT",
  "summary": "One paragraph review summary.",
  "comments": [
    {
      "path": "relative/path/to/file.ts",
      "line": 42,
      "body": "Inline comment text."
    }
  ]
}
```

If you omit the prompt file, a built-in default is used. Custom prompts produce
significantly better, more focused results for your specific stack.

See [`timmy/`'s prompt](https://github.com/modsy/timmy/blob/main/.github/codex/prompts/codex-pr-review.md) as a reference implementation.

### Step 4 — Open a test PR

Push a branch, open a PR, and confirm that:

- The `PR Review` workflow appears in the **Checks** tab.
- A PR review appears in the **Files changed** tab with inline annotations.
- The review summary is stamped with the reviewed commit SHA.

---

## Versioning

**Recommended:** pin to a specific commit SHA for reproducibility and
supply-chain safety.

```yaml
uses: modsy/ci-workflows/.github/workflows/codex-pr-review.yml@<sha>
```

**Acceptable for low-risk teams:** pin to `@main` to always get the latest.

```yaml
uses: modsy/ci-workflows/.github/workflows/codex-pr-review.yml@main
```

---

## Troubleshooting

**Review does not appear**
- Check the Actions run logs for the `codex-review` job.
- Confirm `CODEX_AUTH_JSON` or `CODEX_ACCESS_TOKEN` is set and scoped to the repo.
- Check the "Authenticate Codex" step for error messages.

**401 from api.openai.com/v1/responses**
- Your token is routing to the OpenAI API directly but lacks `api.responses.write` scope.
- Use `CODEX_AUTH_JSON` (full auth.json with refresh_token) instead of `CODEX_ACCESS_TOKEN`.
- The full auth.json causes Codex to route through `chatgpt.com`, which has the correct scope.

**"invalid agent identity JWT format"**
- The value stored in `CODEX_ACCESS_TOKEN` is not a valid JWT or PAT format.
- Use `CODEX_AUTH_JSON` with the full auth.json blob instead.

**"Cannot resolve a PR number"**
- When calling via `workflow_dispatch`, you must pass `pr_number` as an input.

**Fork PRs are not reviewed**
- Intentional for repos using `pull_request_target`. Fork PRs are blocked by the
  fork guard to prevent secret exfiltration. A maintainer can manually trigger a
  review via `workflow_dispatch` after inspecting the fork.

**`pre_install_command` fails**
- Installation failures are `continue-on-error`. The prompt reports the outcome
  so Codex knows which tools are available. Fix the command and push a new commit.
- For monorepos with workspace packages, build them: `npm ci && npm run build --workspaces --if-present`.

**bubblewrap / sandbox errors**
- The workflow automatically runs `sysctl -w kernel.unprivileged_userns_clone=1`
  for Linux runners. This is required for Codex's bubblewrap sandbox.

---

## Contributing

Maintained by the modsy platform team. To propose a change that benefits all
consuming teams, open a PR here. Breaking changes to inputs or the review posting
behavior require a migration note in the PR description.
