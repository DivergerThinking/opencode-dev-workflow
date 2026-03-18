---
name: resolve-issue
description: Protocol for resolving a GitHub issue end-to-end: analyze, plan, implement, test, commit, and open a PR.
---

# Resolve Issue — Protocol

This skill defines the exact steps, conventions, and expectations for the `issue-flow` agent when working through a GitHub issue from start to finish.

---

## Step 0 — Request the Issue URL

Ask the user for the GitHub issue URL. Accept either of these formats:

- Full URL: `https://github.com/<owner>/<repo>/issues/<number>`
- Short reference: `#<number>` (resolves against the current repo's remote)

Extract the issue number from the URL.

Fetch the issue data using the GitHub CLI:

```bash
gh issue view <number> --json number,title,body,labels,comments,assignees,state
```

Display a summary to the user:

```
Issue #<number>: <title>
Labels: <labels>
State: <state>

<body>
```

**Pause and ask the user:** "Issue loaded. Shall I proceed with the analysis?"

---

## Step 1 — Analyze & Plan (architect subagent)

Invoke the `architect` subagent with the following context:

```
Analyze the following GitHub issue for this project and produce a complete implementation plan.

Issue #<number>: <title>

<body>

<comments if any>

Instructions:
- Load all relevant skills before starting.
- Explore the codebase to understand the current architecture.
- Produce your response in the standard format:
  1. Analysis
  2. Proposal
  3. Implementation Plan (ordered, actionable steps with file paths)
  4. Trade-offs
```

After the `architect` returns its plan, display the full plan to the user.

**Pause and ask the user:** "Does the plan look good? You can ask me to adjust it before we proceed."

Wait for the user to approve or refine the plan before continuing.

---

## Step 2 — Implement (general subagent)

Before invoking implementation:

1. Create a branch with the naming convention:

   ```bash
   git checkout -b issue-<number>-<slug>
   ```

   Where `<slug>` is the issue title lowercased, with spaces replaced by hyphens and special characters removed. Max 50 chars total.

   Example: Issue #42 "Add pagination to items endpoint" → `issue-42-add-pagination-to-items-endpoint`

2. Invoke the `general` subagent with the approved plan:

   ```
   Implement the following plan. Load any relevant skills before starting.

   <approved plan from architect>

   Important:
   - Follow all conventions in AGENTS.md.
   - Load the relevant skills (e.g., implement-api-resource, test-api-resource) before writing code.
   - Do not commit changes yet.
   ```

After the `general` subagent finishes, summarize the files changed.

**Pause and ask the user:** "Implementation complete. Review the changes above. Shall I proceed to run tests and linting?"

---

## Step 3 — Test & Lint

Run linting and tests in order:

```bash
make lint
```

```bash
make test
```

If any checks fail:
- Show the errors to the user.
- Automatically iterate and fix the issues (invoke `general` again if needed).
- Re-run until all checks pass.
- Do not ask for approval on each iteration — fix silently and re-run.

Once all checks pass, display:

```
Lint: PASSED
Tests: PASSED
```

**Pause and ask the user:** "All checks passed. Shall I commit the changes and open a Pull Request?"

---

## Step 4 — Commit & Pull Request

### Commit

Stage all changes and create a commit:

```bash
git add .
git commit -m "<type>: resolves #<number> - <title>

Closes #<number>"
```

Choose `<type>` based on the issue content:
- `feat` — new feature or endpoint
- `fix` — bug fix
- `refactor` — code restructuring without behavior change
- `test` — test additions or fixes
- `docs` — documentation only
- `chore` — tooling, dependencies, config

### Pull Request

Open a PR using the GitHub CLI:

```bash
gh pr create \
  --title "<type>: resolves #<number> - <title>" \
  --body "$(cat <<'EOF'
## Summary

<1-3 bullet points describing what was changed and why>

## Changes

<list of key files changed>

## Testing

- [ ] `make lint` passes
- [ ] `make test` passes

Closes #<number>
EOF
)"
```

Display the PR URL to the user.

---

## Conventions Reference

| Convention | Detail |
|---|---|
| Branch name | `issue-<number>-<slug>` (max 60 chars) |
| Commit format | `<type>: resolves #<number> - <title>` |
| PR title | Same as commit message |
| PR body | Summary + Changes + Testing checklist + `Closes #<number>` |
| Test command | `make test` |
| Lint command | `make lint` |
| Null UUID for tests | `00000000-0000-0000-0000-000000000000` |

## Skills to load during implementation

The `general` subagent should check available skills and load those relevant to the issue. Common ones for this project:

| Skill | When to load |
|---|---|
| `implement-api-resource` | Issue involves creating a new API resource/endpoint |
| `test-api-resource` | Issue involves writing or fixing tests for an API resource |
| `resolve-issue` | Always load this skill at the start of the flow |
