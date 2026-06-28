# ci-workflows

Shared reusable GitHub Actions workflows for Lennar engineering teams.

This repo is **public** so any org (`modsy`, `product-org-len`, and future orgs) can call these workflows without cross-org secrets or enterprise sharing agreements.

---

## Workflows

### `codex-pr-review.yml`

Runs an AI code review on every pull request using the Codex CLI. Posts findings as a PR comment marked `<!-- codex-reviewer -->`.

#### How it works

1. Caller repo triggers on `pull_request` or `pull_request_target`.
2. Caller's thin workflow calls `modsy/ci-workflows/.github/workflows/codex-pr-review.yml@main`.
3. The reusable workflow checks out the code, installs Codex CLI, reads a prompt file from the caller repo (if present), and runs `codex exec --ephemeral --sandbox workspace-write`.
4. Codex posts the review as a PR comment.

#### Security model

- **Secrets stay per-org.** `CODEX_ACCESS_TOKEN` is an org-level secret in each org. It is never stored here.
- **Prompt stays private.** The review prompt lives in each service repo at `.github/codex/prompts/codex-pr-review.md`. It is never in this shared repo.
- **`pull_request_target` safety.** Callers using `pull_request_target` MUST include a fork guard (see caller stub below). Without it, fork PRs can exfiltrate secrets.
- **Ephemeral + sandboxed.** Codex runs with `--ephemeral --sandbox workspace-write` — no persistent state, no network access beyond what workspace-write allows.

---

## Adding this to your repo

### Step 1: Add the org-level secret

In your GitHub org settings, add an org secret named `CODEX_ACCESS_TOKEN` with your Anthropic API key. Scope it to the repos that need it.

- **modsy**: Settings > Secrets and variables > Actions > New org secret
- **product-org-len**: Same path

### Step 2: Add the caller workflow

Create `.github/workflows/pr-review.yml` in your service repo:

```yaml
name: PR Review

on:
  pull_request:
    types: [opened, synchronize, reopened]
  # For repos that receive external fork PRs, use pull_request_target instead:
  # pull_request_target:
  #   types: [opened, synchronize, reopened]

jobs:
  # Fork guard — REQUIRED if using pull_request_target.
  # Remove if using pull_request (no fork access to secrets anyway).
  check-fork:
    runs-on: ubuntu-latest
    steps:
      - name: Block fork PRs
        if: github.event.pull_request.head.repo.fork == true
        run: |
          echo "Fork PRs are not reviewed automatically for security reasons."
          exit 1

  codex-review:
    needs: check-fork
    uses: modsy/ci-workflows/.github/workflows/codex-pr-review.yml@main
    secrets:
      CODEX_ACCESS_TOKEN: ${{ secrets.CODEX_ACCESS_TOKEN }}
```

### Step 3: Add your prompt file (optional but recommended)

Create `.github/codex/prompts/codex-pr-review.md` in your repo. This file is your private review instructions — Codex reads it at runtime. If you omit it, a sensible default is used.

Example for a Django/React stack:

```markdown
You are reviewing a pull request in a production web application. The stack is Django (Python) on the backend and React/TypeScript on the frontend.

Review the diff for:
- Bugs and regressions
- Security issues (SQL injection, XSS, hardcoded secrets, OWASP Top 10)
- Performance problems (N+1 queries, missing indexes, large bundle imports)
- API contract breaks (removed fields, renamed endpoints, type changes)
- Missing test coverage for changed behavior

Post a concise markdown summary as a PR comment. Start the comment with:
<!-- codex-reviewer -->

Keep it to the point. Flag blockers clearly. Group findings by severity: Blocking, Suggestion, Nitpick.

Do not use `gh` CLI directly. Output your review as markdown text only.
```

---

## For Vincent (product-org-len teams)

If you have an existing `codex-pr-review.yml` in your repo, here is how to swap it for this shared version:

1. **Delete** your local `.github/workflows/codex-pr-review.yml`.
2. **Add** the caller stub above as `.github/workflows/pr-review.yml` (or keep the same filename — your choice).
3. **Keep** your `.github/codex/prompts/codex-pr-review.md` exactly as-is. The shared workflow reads it from your repo at runtime.
4. **Add** `CODEX_ACCESS_TOKEN` as an org-level secret in `product-org-len` (if it is not already there as a repo secret under a different name — just rename or re-add it at org scope).
5. Open a test PR and confirm the review comment appears.

You do not need to change your prompt, your token, or anything else. The only thing that changes is where the workflow logic lives.

---

## Versioning

Pin to `@main` for latest. To pin to a specific commit for stability:

```yaml
uses: modsy/ci-workflows/.github/workflows/codex-pr-review.yml@<sha>
```

---

## Contributing

This repo is maintained by the modsy/engineering team. Open an issue or PR here to propose changes that benefit all teams.
