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
| [`codex-pr-review.yml`](.github/workflows/codex-pr-review.yml) | AI code review on every PR using the Codex CLI. Posts a structured finding comment. |

---

## `codex-pr-review` — AI PR Review

### How it works

1. A service repo's caller workflow triggers on `pull_request` or `pull_request_target`.
2. The caller invokes this shared workflow via `uses: modsy/ci-workflows/.github/workflows/codex-pr-review.yml@main`.
3. The shared workflow:
   - Checks out the PR's merge commit (`refs/pull/N/merge`) with `persist-credentials: false`.
   - Fetches PR metadata and any linked issues via the `gh` CLI.
   - Optionally runs a `pre_install_command` to install project dependencies.
   - Reads the caller repo's prompt from `.github/codex/prompts/codex-pr-review.md` (or uses a built-in default).
   - Runs `codex exec --ephemeral --sandbox workspace-write`.
   - Posts the review as a new PR comment marked `<!-- codex-reviewer -->`.

### Security model

| Concern | How it is addressed |
|---|---|
| Secrets isolation | `CODEX_ACCESS_TOKEN` is an org-level secret in each org, passed to the shared workflow at call time. It is never stored in this repo. |
| Fork PR exfiltration | Callers using `pull_request_target` MUST use the fork guard shown in the examples (an `if:` on the job, not a separate prerequisite job). The guard restricts automatic runs to same-repo PRs. |
| Prompt privacy | The review prompt lives in each service repo at `.github/codex/prompts/codex-pr-review.md`. It is read at runtime and never sent to this shared repo. |
| Sandboxed execution | Codex runs with `--ephemeral --sandbox workspace-write`. No persistent state; filesystem writes are confined to the workspace. |
| Credential leak via git | `persist-credentials: false` on checkout removes the `GITHUB_TOKEN` from the git config, preventing Codex from using it to push or fetch. |
| Script injection | All GitHub context values are passed to shell via environment variables, never interpolated directly into `run:` scripts. |
| Supply-chain pinning | The Codex CLI is installed at an explicit version (`codex_version` input, defaults to `0.142.0`). Callers should pin and bump deliberately. |

### Inputs

| Input | Type | Required | Default | Description |
|---|---|---|---|---|
| `pr_number` | number | No | auto | PR number to review. Auto-resolved from the event. Pass explicitly for `workflow_dispatch`. |
| `codex_version` | string | No | `0.142.0` | `@openai/codex` npm version to install. Pin and bump deliberately. |
| `pre_install_command` | string | No | `""` | Shell command to run before Codex. Use to pre-install dependencies (e.g. `npm ci`, `pip install -r requirements.txt`). Runs with `continue-on-error`; outcome is reported in the prompt. |

### Secrets

| Secret | Required | Description |
|---|---|---|
| `CODEX_ACCESS_TOKEN` | Yes | Claude Code access token (starts with `at-`) or agent-identity JWT. Obtain from claude.ai account settings. Store as an org-level secret scoped to the repos that use it. |

### Outputs

| Output | Description |
|---|---|
| `comment_url` | URL of the posted Codex review comment. |

---

## Adoption guide

### Step 1 — Add the org-level secret

In your GitHub org settings, add an org secret named **`CODEX_ACCESS_TOKEN`**
with your Claude Code access token (starts with `at-`). Obtain it from
[claude.ai](https://claude.ai) account settings under API access. Scope it to
the repos that need it.

- **modsy**: `Settings > Secrets and variables > Actions > New org secret`
- **product-org-len**: Same path.

If you already have the key as a repo-level secret, you can promote it to org
scope or leave it at repo scope — both work.

### Step 2 — Add the caller workflow

Pick the template that matches your situation:

| Template | Use when |
|---|---|
| [`examples/pr-review-internal.yml`](examples/pr-review-internal.yml) | All contributors have push access to the repo (no fork PRs). Simpler, safer. |
| [`examples/pr-review-with-fork-guard.yml`](examples/pr-review-with-fork-guard.yml) | Repo receives PRs from forks (open source or cross-org). Includes mandatory fork guard. |

Copy the relevant file to `.github/workflows/pr-review.yml` in your service
repo and adjust `pre_install_command` for your stack.

### Step 3 — Add your prompt (recommended)

Create `.github/codex/prompts/codex-pr-review.md` in your repo. This is your
private review instruction set — Codex reads it at runtime.

Start from the template: [`examples/prompt-template.md`](examples/prompt-template.md).

If you omit this file, a sensible built-in default is used, but you will get
better results by tuning the prompt to your stack and standards.

### Step 4 — Open a test PR

Push a branch, open a PR, and confirm that:
- The `PR Review` workflow appears in the Checks tab.
- A `<!-- codex-reviewer -->` comment appears on the PR.
- The comment is stamped with the reviewed commit SHA.

---

## For teams migrating from an existing Codex workflow

If you have an existing self-contained `codex-pr-review.yml` in your repo,
the migration is minimal:

1. **Keep your prompt.** Your `.github/codex/prompts/codex-pr-review.md` is
   read from your repo at runtime. No changes needed.

2. **Keep your secret.** If `CODEX_ACCESS_TOKEN` is already set as a repo
   secret, it works as-is. Optionally promote it to org scope so other repos
   can share it without duplicating it.

3. **Replace your workflow file.** Delete your existing workflow file and drop
   in the appropriate caller template from `examples/`. The shared workflow
   already handles: PR metadata fetching, linked issue resolution, SHA stamping,
   comment truncation, job summary, and bubblewrap setup.

4. **Add `pre_install_command` if you pre-installed deps.** If your old
   workflow ran `npm ci` or `pip install` before calling Codex, move that
   command into the `pre_install_command` input.

5. Open a test PR to confirm.

Nothing in your prompt, token, or project needs to change. Only the workflow
orchestration moves here.

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

Never pin to a mutable tag (`@v1`) without a corresponding branch protection
rule — mutable tags can be force-pushed.

---

## Troubleshooting

**Review comment does not appear**
- Check the Actions run logs for the `codex-review` job.
- Confirm `CODEX_ACCESS_TOKEN` is set and scoped to the repo.
- Confirm the caller passes `secrets: CODEX_ACCESS_TOKEN: ${{ secrets.CODEX_ACCESS_TOKEN }}`.

**"Unable to resolve a PR number"**
- When calling via `workflow_dispatch`, you must pass `pr_number` as an input.

**Fork PRs are not reviewed**
- This is intentional for repos using `pull_request_target`. Fork PRs are
  blocked by the fork guard to prevent secret exfiltration. A maintainer can
  manually trigger a review via `workflow_dispatch` after inspecting the fork.

**Review runs on draft PRs**
- The caller templates skip drafts via the `if:` condition. If you copied an
  older version without the draft check, add:
  `github.event.pull_request.draft == false` to your job `if:`.

**`pre_install_command` fails**
- Installation failures are `continue-on-error`. The Codex prompt reports the
  outcome so Codex knows which tools are available. Fix the command in your
  caller workflow and push.

**Bubblewrap / namespace errors on the runner**
- The workflow enables unprivileged user namespaces via `sysctl`. This works
  on standard GitHub-hosted `ubuntu-latest` runners. Self-hosted runners may
  need these sysctls set at the OS level.

---

## Contributing

This repo is maintained by the modsy platform team. To propose a change that
benefits all consuming teams, open an issue or pull request here. Breaking
changes to inputs or the `<!-- codex-reviewer -->` marker require a migration
note in the PR description.
