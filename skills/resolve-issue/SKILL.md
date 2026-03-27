---
name: resolve-issue
description: Protocol for resolving a GitHub issue end-to-end: analyze, plan, implement, test, commit, and open a PR.
---

# Resolve Issue — Protocol

This skill defines the exact steps, conventions, and expectations for the `issue-flow` agent when working through a GitHub issue from start to finish.

## Issue Tracking Convention

Throughout the flow, the agent **keeps the GitHub issue updated** at every phase transition. This means:

- **Assignment:** assign `@me` at the start; unassign at the end if the issue is closed via PR.
- **Progress comments:** post a structured comment on the issue at the end of each phase, summarising what was done and what comes next.
- **Closure:** the issue is closed automatically when the PR merges (via `Closes #<number>` in the PR body), so no explicit close command is needed — but a final comment is posted to record the PR link.

The comment format for each phase update is:

```
## [Phase <N> — <phase_name>] ✓

<one-paragraph summary of what was done in this phase>

**Next step:** <what happens in Phase N+1>
```

This creates a clear, auditable trail directly on the GitHub issue for anyone following along.

---

## Step -1 — Resume Detection

Before doing anything else, check whether a history file already exists for this issue in the target repository:

```bash
cat .opencode/history/issue-<number>.md
```

Run this command exactly as shown — do not add `2>/dev/null`, `|| echo`, `&&`, or any shell operators. If the command fails (file not found), that is your signal to proceed normally to Step 0.

If the file **exists and the command succeeds**, read its front-matter to determine the last completed phase (`phase` field) and display a resume prompt to the user:

```
Found an existing session for issue #<number> — "<title>".
Last completed phase: <phase> — <phase_name>

<brief summary of what was recorded in each completed phase>

Resume from Phase <phase+1>, or start over?
```

If the user chooses **resume**:
- Load all context recorded in the history file (issue data, architect plan, branch name, decisions, etc.) directly from the file — do not re-fetch or re-run what was already done.
- Jump to the next phase after the last completed one, reinjecting the saved context into any subagent prompts.

If the user chooses **start over**:

1. **Fetch all existing progress comments** posted by the bot on the issue — these are the `## [Phase N — ...]` comments from the previous session. Use `gh issue view` with `--json comments` and filter for comments whose body starts with `## [Phase`:

   ```bash
   gh issue view <number> --json comments --jq '.comments[] | select(.body | startswith("## [Phase")) | .databaseId'
   ```

2. **Add a 👎 reaction to each** of those comment IDs. This marks them as voided directly on GitHub without removing the audit trail:

   ```bash
   gh api repos/{owner}/{repo}/issues/comments/<comment_id>/reactions \
     --method POST \
     -f content="-1"
   ```

   Repeat for every comment ID returned in step 1.

3. **Post a reset announcement comment** on the issue so the timeline is clear:

   ```bash
   gh issue comment <number> --body "## [Session Reset]

   A previous flow session for this issue has been discarded and a fresh session is starting. The prior phase comments above have been marked as voided (👎)."
   ```

4. **Delete the existing history file.**

5. Proceed normally from Step 0.

---

## History File

The history file is the source of truth for the state of a running issue session. It is stored in the **target repository** (not in the opencode config repo) at:

```
.opencode/history/issue-<number>.md
```

### Format

The file uses YAML front-matter followed by Markdown sections, one per completed phase:

```markdown
---
issue: <number>
title: "<issue title>"
branch: issue-<number>-<slug>
phase: <last completed phase number>
phase_name: "<last completed phase name>"
last_updated: "<ISO 8601 timestamp>"
---

## Phase 0 — Issue Loaded
...

## Phase 1 — Triage & Scope
...

## Phase 2 — Analyze & Plan
...
```

### Usage tracking

At the end of each phase section, append a `### Usage` block recording the model and token counts for that phase:

```markdown
### Usage
- **Model:** <model-id as reported by your session context, e.g. `anthropic/claude-sonnet-4-20250514`>
- **Input tokens:** <input token count for this phase's work>
- **Output tokens:** <output token count for this phase's work>
```

**How to obtain these values:**

- **Model:** Use the model ID you are currently running under. This is available from your session metadata. If you are unsure, use the model ID configured in the agent's front-matter or the one reported by your provider.
- **Token counts:** opencode does not expose a programmatic token API to agents. Use the cumulative token counters visible in the UI at the start and end of the phase, or estimate based on the context size if the UI values are unavailable. Always record what you actually know; never fabricate precise numbers — if you cannot determine them, write `unknown`.

These values are recorded **per phase**, not cumulatively, so the reader can see the cost of each step independently.

### Initialization

Before writing the first history entry, ensure `.opencode/history/` is excluded from git in the target repository.

Use the **Read tool** (not bash) to read `.gitignore` and check whether it already contains `.opencode/` or `.opencode/history/`. Do not use `grep` or any shell compound command for this check.

If `.opencode/history/` is not present, append it using bash:

```bash
echo '.opencode/history/' >> .gitignore
```

Then create the directory:

```bash
mkdir -p .opencode/history
```

Run each of these bash commands independently — never chain them with `&&`, `||`, or `;`.

### Update cadence

Write or update the history file **at the end of each phase**, immediately before the pause point. Each write overwrites the entire file (update the front-matter fields and append the new phase section).

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

### Write history — Phase 0

Initialize (or overwrite) the history file with the following content:

```markdown
---
issue: <number>
title: "<title>"
branch: ""
phase: 0
phase_name: "Issue Loaded"
last_updated: "<ISO 8601 timestamp>"
---

## Phase 0 — Issue Loaded
- **Issue:** #<number>
- **Title:** <title>
- **State:** <state>
- **Labels:** <labels>
- **Assignees:** <assignees or "none">

### Body
<full issue body>

### Comments
<comments if any, or "No comments.">

### Usage
- **Model:** <model-id>
- **Input tokens:** <count or "unknown">
- **Output tokens:** <count or "unknown">
```

### Update issue — Phase 0

Assign the issue to yourself and post a comment to signal that work has started:

```bash
gh issue edit <number> --add-assignee "@me"
```

```bash
gh issue comment <number> --body "## [Phase 0 — Issue Loaded] ✓

Issue has been picked up and is now in active development.

**Next step:** Triage, requirements analysis, scope definition, and impact assessment."
```

**Pause and ask the user:** "Issue loaded. Shall I proceed with triage and scope analysis?"

---

## Step 1 — Triage & Scope

This phase characterises the issue before any architectural analysis begins. It is executed by the **`triage` subagent**, which works in a clean, minimal context. The primary agent (you) then presents the report to the user, handles any clarifications, applies labels, and manages the pause point.

### 1.1 — Fetch labels and invoke the triage subagent

First, collect the data the subagent needs:

```bash
gh issue view <number> --json number,title,body,labels,comments,assignees,state
gh label list --json name,description --limit 100
```

Then invoke the `triage` subagent, passing all the collected data:

```
Issue to triage:

Number: <number>
Title: <title>
State: <state>
Existing labels: <labels>

Body:
<full issue body>

Comments:
<comments if any, or "No comments.">

Repository labels available:
<full output of gh label list>
```

The subagent will return a structured **Triage Report**. Do not proceed until it responds.

### 1.2 — Present the report and handle ambiguities

Display the full Triage Report to the user exactly as returned by the subagent.

If the report lists any **Ambiguities**, present them clearly and wait for the user to resolve them before continuing. Update the report with the user's answers before writing the history.

### 1.3 — Apply labels on the GitHub issue

Once the user has reviewed the report (and resolved any ambiguities), apply the labels:

```bash
gh label create "<type-label>" --color "<hex>" --description "<description>" 2>/dev/null || true
gh issue edit <number> --add-label "<type-label>"
```

Default colours if creating labels:
- `documentation` → `#0075ca`
- `enhancement` → `#a2eeef`
- `bug` → `#d73a4a`
- `fix` → `#e4e669`
- `chore` → `#ededed`

Remove any pre-existing type labels that conflict with the determined type.

Apply the `in-analysis` label to signal active triage:

```bash
gh label create "in-analysis" --color "#fbca04" --description "Issue is being actively analysed" 2>/dev/null || true
gh issue edit <number> --add-label "in-analysis"
```

### 1.4 — Append acceptance criteria to the issue body (if derived)

If the Triage Report indicates the acceptance criteria were **derived** (not extracted from the issue), update the issue body to make them permanently visible on GitHub:

```bash
gh issue edit <number> --body "$(gh issue view <number> --json body --jq '.body')

---

## Acceptance Criteria

<acceptance criteria list from the report>"
```

### Write history — Phase 1

Update the history file: set `phase: 1`, `phase_name: "Triage & Scope"`, update `last_updated`, and append:

```markdown
## Phase 1 — Triage & Scope

### Classification
- **Issue type:** <type>
- **Labels applied:** <list>

### Requirements
#### Functional
<list>

#### Non-functional
<list>

#### Acceptance criteria
- **Source:** <"Extracted from issue" or "Derived by triage agent — added to issue body">
<list>

#### Ambiguities
<list or "None identified.">

### Scope
#### In scope
<list>

#### Out of scope
<list>

### Impact Analysis
#### Direct impact
<list>

#### Indirect impact
<list>

#### Data impact
<list or "None.">

#### API impact
<list or "None.">

#### Test impact
<list>

### User Decision
<"Approved as-is." or "Adjusted: <description of changes>">

### Usage
- **Model:** <model-id>
- **Input tokens:** <count or "unknown">
- **Output tokens:** <count or "unknown">
```

### Update issue — Phase 1

Post a comment with the triage results:

```bash
gh issue comment <number> --body "## [Phase 1 — Triage & Scope] ✓

**Type:** <type> (<label applied>)

**Acceptance criteria:** <'Extracted from issue' or 'Derived by triage agent and added to issue body'>

**Scope:**
<in-scope bullet points>

**Impact areas:**
<direct and indirect impact bullet points>

**Ambiguities to resolve:** <count or 'None'>

**Next step:** Architectural analysis and implementation planning."
```

**Pause and ask the user:** "Triage complete. Does the scope and impact analysis look correct? Shall I proceed with the architectural plan?"

Wait for the user to approve before continuing.

---

## Step 2 — Analyze & Plan (architect subagent)

Invoke the `architect` subagent with the following context:

```
Analyze the following GitHub issue for this project and produce a complete implementation plan.

Issue #<number>: <title>

<body>

<comments if any>

Triage results:
- Type: <type>
- Scope: <in-scope items>
- Impact areas: <direct and indirect impact>

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

### Write history — Phase 2

Update the history file: set `phase: 2`, `phase_name: "Analyze & Plan"`, update `last_updated`, and append:

```markdown
## Phase 2 — Analyze & Plan

### Architect Output
<full plan as returned by the architect subagent>

### User Decision
<"Approved as-is." or "Adjusted: <description of the user's requested changes>">

### Usage
- **Model:** <model-id>
- **Input tokens:** <count or "unknown">
- **Output tokens:** <count or "unknown">
```

### Update issue — Phase 2

Post a comment summarising the approved plan:

```bash
gh issue comment <number> --body "## [Phase 2 — Analyze & Plan] ✓

The implementation plan has been reviewed and approved.

**Plan summary:**
<3-5 bullet points distilling the key steps from the architect's plan>

**Next step:** Creating branch \`issue-<number>-<slug>\` and beginning implementation."
```

**Pause and ask the user:** "Does the plan look good? You can ask me to adjust it before we proceed."

Wait for the user to approve or refine the plan before continuing.

---

## Step 3 — Implement (general subagent)

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

### Write history — Phase 3

Update the history file: set `phase: 3`, `phase_name: "Implementation"`, `branch: issue-<number>-<slug>`, update `last_updated`, and append:

```markdown
## Phase 3 — Implementation
- **Branch:** issue-<number>-<slug>
- **Files changed:**
<list each file, one per line, prefixed with "  -">

### Summary
<summary as reported by the general subagent>

### Usage
- **Model:** <model-id>
- **Input tokens:** <count or "unknown">
- **Output tokens:** <count or "unknown">
```

### Update issue — Phase 3

Post a comment listing the files changed during implementation:

```bash
gh issue comment <number> --body "## [Phase 3 — Implementation] ✓

Implementation is complete on branch \`issue-<number>-<slug>\`.

**Files changed:**
<list each file, one per line, prefixed with '- '>

**Next step:** Running \`make lint\` and \`make test\`."
```

**Pause and ask the user:** "Implementation complete. Review the changes above. Shall I proceed to run tests and linting?"

---

## Step 4 — Test & Lint

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
- Track the number of fix iterations for the history.

Once all checks pass, display:

```
Lint: PASSED
Tests: PASSED
```

### Write history — Phase 4

Update the history file: set `phase: 4`, `phase_name: "Tests & Lint"`, update `last_updated`, and append:

```markdown
## Phase 4 — Tests & Lint
- **Lint:** PASSED
- **Tests:** PASSED
- **Fix iterations required:** <number, or "0 — passed on first run">

<If fix iterations > 0, briefly describe what was fixed>

### Usage
- **Model:** <model-id>
- **Input tokens:** <count or "unknown">
- **Output tokens:** <count or "unknown">
```

### Update issue — Phase 4

Post a comment with the lint and test results:

```bash
gh issue comment <number> --body "## [Phase 4 — Tests & Lint] ✓

All quality checks have passed.

- **Lint:** PASSED
- **Tests:** PASSED
- **Fix iterations required:** <number, or '0 — passed on first run'>

**Next step:** Committing changes and opening a Pull Request."
```

**Pause and ask the user:** "All checks passed. Shall I commit the changes and open a Pull Request?"

---

## Step 5 — Commit & Pull Request

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

### Write history — Phase 5

Update the history file: set `phase: 5`, `phase_name: "Commit & PR"`, update `last_updated`, and append:

```markdown
## Phase 5 — Commit & PR
- **Commit type:** <type>
- **Commit message:** `<type>: resolves #<number> - <title>`
- **PR URL:** <url>

### Usage
- **Model:** <model-id>
- **Input tokens:** <count or "unknown">
- **Output tokens:** <count or "unknown">
```

### Update issue — Phase 5

Post a final comment on the issue linking the PR:

```bash
gh issue comment <number> --body "## [Phase 5 — Pull Request Opened] ✓

A Pull Request has been opened for review.

- **PR:** <pr-url>
- **Commit:** \`<type>: resolves #<number> - <title>\`

The issue will be closed automatically when the PR is merged."
```

### Post history as PR comment

After the PR is created, read the full history file and post it as a comment on the PR:

```bash
gh pr comment <pr-number> --body "$(cat <<'EOF'
<details>
<summary>Issue Flow History — #<number></summary>

<full content of .opencode/history/issue-<number>.md, excluding the YAML front-matter block>

</details>
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
| History file | `.opencode/history/issue-<number>.md` in the target repo |
| History in git | Excluded via `.gitignore` — never committed |

## Skills to load during implementation

The `general` subagent should check available skills and load those relevant to the issue. Common ones for this project:

| Skill | When to load |
|---|---|
| `implement-api-resource` | Issue involves creating a new API resource/endpoint |
| `test-api-resource` | Issue involves writing or fixing tests for an API resource |
| `resolve-issue` | Always load this skill at the start of the flow |
