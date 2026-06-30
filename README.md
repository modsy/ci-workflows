# ci-workflows

Shared, reusable GitHub Actions workflows for Lennar engineering teams.

The repo is **public**, so any repo in any org can call these workflows directly — no
cross-org setup. The review logic lives here and is maintained once; consuming repos add
a tiny caller and nothing else. No secrets are stored here.

---

## `codex-pr-review` — AI code review on every PR

Automated, codebase-aware code review powered by OpenAI Codex. It behaves like GitHub
Copilot's reviewer — inline diff comments plus a summary — with a few deliberate
improvements:

- **Zero config.** Add one ~20-line caller. No prompt, no stack hints, no install script.
  The workflow auto-detects your stack (npm/yarn/pnpm, uv/pip, go) and installs deps.
- **Codebase-aware.** A single stack-agnostic prompt (bundled here) detects the language
  and reads your repo's own `AGENTS.md` / `CLAUDE.md` / `.agents/rules/` for conventions.
- **Never blocks.** The review is always posted as a non-blocking `COMMENT`. The model's
  verdict is shown as advisory text (`Verdict: Changes suggested — 2 issues · 1 suggestion`),
  so resolving threads is clean and nothing wedges your merge.
- **Consistent comments.** [Conventional Comments](https://conventionalcomments.org)
  (`issue` / `suggestion` / `nitpick`), each with a one-line subject, a required *why*,
  and a concrete fix. No emoji.
- **Maintained centrally.** Change the review behavior for every repo by editing the
  prompt here — not by touching each consumer.

---

## Quick start (2 steps)

### 1. Add the auth secret

Add **`CODEX_AUTH_JSON`** at the **org** level (one-time, shared by all repos) under
*Settings → Secrets and variables → Actions*. Its value is the full contents of
`~/.codex/auth.json` from a machine where you've run `codex auth`.

> For Lennar ChatGPT Business/Enterprise accounts, use `CODEX_AUTH_JSON` (not
> `CODEX_ACCESS_TOKEN`): its refresh token lets Codex authenticate through `chatgpt.com`.

### 2. Add the caller workflow

Create `.github/workflows/pr-review.yml` — this is the **entire** integration, identical
in every repo:

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
    # Same-repo guard: never runs for fork PRs, keeping credentials away from
    # untrusted code. (Don't add workflow_dispatch — an arbitrary PR number
    # would bypass this guard.)
    if: >-
      github.event.pull_request.head.repo.full_name == github.repository &&
      github.event.pull_request.draft == false
    uses: modsy/ci-workflows/.github/workflows/codex-pr-review.yml@main
    with:
      pr_number: ${{ github.event.pull_request.number }}
    secrets:
      CODEX_AUTH_JSON: ${{ secrets.CODEX_AUTH_JSON }}
```

That's it. Open a PR and the reviewer comments within a minute or two. **Do not add a
prompt** — it's bundled here and applied to every caller.

---

## How it works

1. Your caller triggers on `pull_request` and calls this reusable workflow.
2. The workflow:
   - Checks out the PR merge commit (`persist-credentials: false`, so the token can't
     push or fetch).
   - Fetches PR metadata, the diff, and any linked issues for context.
   - **Auto-installs dependencies** by detecting a root manifest — `package-lock.json` →
     `npm ci` (and builds workspaces if it's a monorepo), `yarn.lock` / `pnpm-lock.yaml`,
     `uv.lock` → `uv sync`, `requirements.txt` / `pyproject.toml` → pip, `go.mod` →
     `go mod download`. Best-effort and `continue-on-error`: if it can't install, the
     review still runs statically.
   - Loads the review prompt from **this repo** (version-locked to the workflow's own
     commit) — never from the PR under review.
   - Runs the Codex CLI in an ephemeral sandbox and parses its JSON output.
   - Posts a single non-blocking `COMMENT` review with inline diff comments.

---

## Inputs

| Input | Required | Default | Description |
|---|---|---|---|
| `pr_number` | No | auto | PR to review. Resolved from the triggering event; the caller above passes it explicitly. |
| `codex_version` | No | `0.142.0` | Pinned `@openai/codex` version. |
| `pre_install_command` | No | `""` | Optional **override** for dependency setup. Leave empty for auto-detect. Set only for non-standard layouts (e.g. a package nested in a monorepo subdir). |

## Secrets

Provide **one** (org-level recommended):

| Secret | When to use |
|---|---|
| `CODEX_AUTH_JSON` | **Preferred.** Full `~/.codex/auth.json`; includes the refresh token so Codex routes through `chatgpt.com`. |
| `CODEX_ACCESS_TOKEN` | A personal access token (`at-*`), OpenAI API key (`sk-*`), or ChatGPT OAuth JWT. |

## Outputs

| Output | Description |
|---|---|
| `review_url` | URL of the posted PR review. |

---

## Customizing reviews

There is **one** prompt for all repos:
[`.github/codex/prompts/codex-pr-review.md`](.github/codex/prompts/codex-pr-review.md).

- **Change it for everyone:** edit the prompt here and open a PR.
- **Tune it for one repo:** document the convention in that repo's `AGENTS.md` /
  `CLAUDE.md` — the prompt reads them at review time.

You should never need a prompt file in a consuming repo.

---

## Security model

| Concern | How it's addressed |
|---|---|
| Secret isolation | Auth secrets are stored per org/repo and passed at call time. Never stored here. |
| Prompt integrity | The prompt is read from this repo at the workflow's own commit, never from the PR — a PR can't change what the credentialed reviewer runs. |
| Fork exfiltration | The caller's same-repo `if:` guard prevents the workflow (and its secrets) from running on fork PRs. |
| Credential leak via git | `persist-credentials: false` keeps `GITHUB_TOKEN` out of git config. |
| Script injection | GitHub context values reach the shell only as environment variables, never interpolated into `run:` blocks. |
| Sandbox | Codex runs `--ephemeral --sandbox workspace-write`: no persistent state, writes scoped to the workspace. |

---

## Versioning

`@main` tracks the latest. Pin to a commit SHA for full reproducibility:

```yaml
uses: modsy/ci-workflows/.github/workflows/codex-pr-review.yml@<sha>
```

## Troubleshooting

| Symptom | Cause / fix |
|---|---|
| Review never appears | Confirm `CODEX_AUTH_JSON` (or `CODEX_ACCESS_TOKEN`) is set and visible to the repo. The `codex-review` job's auth step fails fast without it. |
| `Either CODEX_ACCESS_TOKEN or CODEX_AUTH_JSON secret is required` | No auth secret reached the workflow. Add it at the org or repo level. |
| `401 from api.openai.com/v1/responses` | Token routed to the OpenAI API but lacks `api.responses.write`. Use `CODEX_AUTH_JSON` so Codex routes through `chatgpt.com`. |
| `invalid agent identity JWT format` | `CODEX_ACCESS_TOKEN` isn't a valid JWT/PAT. Use `CODEX_AUTH_JSON`. |
| Review is shallow / misses conventions | Add an `AGENTS.md` or `CLAUDE.md` to your repo; the prompt reads them for project rules. |
| Dependencies not installed | Auto-detect is best-effort. For a nested monorepo package, set `pre_install_command` to your install command. |
| Fork PRs not reviewed | Intentional — the same-repo guard keeps secrets away from fork code. |

## Contributing

Maintained by the modsy platform team. Open a PR for changes that benefit consuming teams,
and call out any breaking input or behavior change in the description.
