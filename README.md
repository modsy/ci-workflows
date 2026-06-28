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
| [`claude-pr-review.yml`](.github/workflows/claude-pr-review.yml) | AI code review on every PR push using the Claude API. Posts a structured PR review with inline diff annotations, identical in behavior to GitHub Copilot code review. |

---

## `claude-pr-review` — AI PR Review

### How it works

1. A service repo's caller workflow triggers on `pull_request` (or
   `pull_request_target` for repos with fork PRs).
2. The caller invokes this shared workflow via
   `uses: modsy/ci-workflows/.github/workflows/claude-pr-review.yml@main`.
3. The shared workflow:
   - Checks out the PR's merge commit (`refs/pull/N/merge`) with
     `persist-credentials: false`.
   - Fetches PR metadata, the full diff, and any linked issues.
   - Optionally runs a `pre_install_command` (e.g. `npm ci`) and reports
     the outcome in the prompt.
   - Reads a custom review prompt from `.github/claude/prompts/pr-review.md`
     in the caller repo (or falls back to a sensible built-in default).
   - Calls the Anthropic Messages API and parses the structured JSON response.
   - Posts the result as a PR review (verdict + inline diff comments) via
     `github.rest.pulls.createReview` — the same API used by GitHub Copilot.

### Security model

| Concern | How it is addressed |
|---|---|
| Secrets isolation | `ANTHROPIC_API_KEY` is stored in each org or repo and passed explicitly at call time. It is never stored here. |
| Fork PR exfiltration | Callers using `pull_request_target` MUST include the fork guard shown in the examples (`if:` on the job; never on a separate prerequisite job). |
| Prompt privacy | The review prompt lives in each caller repo at `.github/claude/prompts/pr-review.md` and is read at runtime. |
| Credential leak via git | `persist-credentials: false` removes `GITHUB_TOKEN` from the git config so scripts cannot use it to push or fetch. |
| Script injection | All GitHub context values are passed to shell via environment variables, never interpolated directly into `run:` scripts. |
| Diff injection | The diff is embedded in the prompt via `jq` string encoding before the API call, not via shell substitution. |

### Inputs

| Input | Type | Required | Default | Description |
|---|---|---|---|---|
| `pr_number` | number | No | auto | PR number to review. Auto-resolved from the event; pass explicitly for `workflow_dispatch`. |
| `model` | string | No | `claude-sonnet-4-6` | Anthropic model ID. Pin this in your caller for reproducible reviews. |
| `max_tokens` | number | No | `4096` | Maximum tokens in the model response. |
| `pre_install_command` | string | No | `""` | Shell command to run before the review (e.g. `npm ci`). Runs with `continue-on-error`; outcome is reported in the prompt. |

### Secrets

| Secret | Required | Description |
|---|---|---|
| `ANTHROPIC_API_KEY` | Yes | Anthropic API key with access to the chosen model. Store as an org-level secret scoped to repos that use it, or as a repo-level secret. |

### Outputs

| Output | Description |
|---|---|
| `review_url` | URL of the posted PR review. |

---

## Adoption guide

### Step 1 — Add the secret

Add **`ANTHROPIC_API_KEY`** as a secret in your org or repo:

- **Org-level** (recommended — shared across all repos that need it):
  `GitHub org > Settings > Secrets and variables > Actions > New org secret`
- **Repo-level**: `Repo > Settings > Secrets and variables > Actions > New repository secret`

The key must have access to the model specified in the `model` input
(default: `claude-sonnet-4-6`). Obtain it from
[console.anthropic.com](https://console.anthropic.com) under API Keys.

### Step 2 — Add the caller workflow

Pick the template that matches your situation:

| Template | Use when |
|---|---|
| [`examples/pr-review-internal.yml`](examples/pr-review-internal.yml) | All contributors have push access to the repo (no fork PRs). Simpler, safer. |
| [`examples/pr-review-with-fork-guard.yml`](examples/pr-review-with-fork-guard.yml) | Repo receives PRs from forks (open source or cross-org). Includes mandatory fork guard. |

Copy the relevant file to `.github/workflows/pr-review.yml` in your service
repo. Adjust `pre_install_command` for your stack.

### Step 3 — Add a custom prompt (recommended)

Create `.github/claude/prompts/pr-review.md` in your repo. This is your
private, stack-specific review instruction set read at runtime.

Start from the template: [`examples/prompt-template.md`](examples/prompt-template.md).

If you omit this file, a built-in default is used. Custom prompts consistently
produce better, more focused results.

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
uses: modsy/ci-workflows/.github/workflows/claude-pr-review.yml@<sha>
```

**Acceptable for low-risk teams:** pin to `@main` to always get the latest.

```yaml
uses: modsy/ci-workflows/.github/workflows/claude-pr-review.yml@main
```

Never pin to a mutable tag (`@v1`) without a corresponding branch protection
rule — mutable tags can be force-pushed.

---

## Troubleshooting

**Review does not appear**
- Check the Actions run logs for the `claude-review` job.
- Confirm `ANTHROPIC_API_KEY` is set and scoped to the repo.
- Confirm the caller passes `secrets: ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}`.

**"Cannot resolve a PR number"**
- When calling via `workflow_dispatch`, you must pass `pr_number` as an input.

**Anthropic API returned HTTP 401**
- The `ANTHROPIC_API_KEY` value is invalid or has been revoked. Rotate the key
  in console.anthropic.com and update the secret.

**Anthropic API returned HTTP 429**
- Rate limit hit. The workflow will fail and GitHub will not auto-retry it. Re-run
  the failed job from the Actions UI once the limit clears.

**Fork PRs are not reviewed**
- Intentional for repos using `pull_request_target`. Fork PRs are blocked by the
  fork guard to prevent secret exfiltration. A maintainer can manually trigger a
  review via `workflow_dispatch` after inspecting the fork.

**Review runs on draft PRs**
- The caller templates skip drafts via the `if:` condition. If you copied an
  older version without the draft check, add
  `github.event.pull_request.draft == false` to your job `if:`.

**`pre_install_command` fails**
- Installation failures are `continue-on-error`. The prompt reports the outcome
  so Claude knows which tools are available. Fix the command and push a new commit.

---

## Contributing

Maintained by the modsy platform team. To propose a change that benefits all
consuming teams, open a PR here. Breaking changes to inputs or the review posting
behavior require a migration note in the PR description.
