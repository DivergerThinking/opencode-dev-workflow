---
name: resolve-issue
description: Protocol for resolving a work specification end-to-end — from a GitHub issue or a direct user description — through triage, analysis, implementation, build, test, commit, and PR.
---

# Resolve Issue — Protocol

This skill defines the exact steps, conventions, and expectations for the `issue-flow` agent when working through a piece of work from start to finish.

## Two modes of operation

The flow supports two **input modes**, determined at Step 0.1:

| Mode | Source of specification | GitHub integration |
|------|-------------------------|--------------------|
| `issue` | GitHub issue (URL or `#number`) | Full: labels, comments, assignment, PR closure |
| `text` | Description written/pasted by the user | None: only local history file is maintained |

The mode is stored in the history file front-matter as `mode: issue` or `mode: text`. All steps that interact with the GitHub issue API are marked **[mode: issue only]** and must be skipped in `text` mode.

---

## Issue Tracking Convention **[mode: issue only]**

When `mode: issue`, the agent **keeps the GitHub issue updated** at every phase transition:

- **Assignment:** assign `@me` at the start.
- **Progress comments:** post a structured comment on the issue at the end of each phase.
- **Closure:** the issue is closed automatically when the PR merges (via `Closes #<number>` in the PR body).

The comment format for each phase update is:

```
## [Phase <N> — <phase_name>] ✓

<one-paragraph summary of what was done in this phase>

**Next step:** <what happens in Phase N+1>
```

When `mode: text`, no GitHub issue comments are posted. The history file is the only audit trail.

---

## Step -1 — Resume Detection

Before doing anything else, check whether a history file already exists for this work item in the target repository. History files live at:

- `mode: issue` → `.opencode/history/issue-<number>.md`
- `mode: text` → `.opencode/history/session-<slug>.md`

Since you don't know the mode yet, scan for any recent history file:

```bash
cat .opencode/history/issue-<number>.md
```

If the user already provided a number or identifier, use it. Otherwise, list existing history files:

```bash
ls .opencode/history/
```

Run each command exactly as shown — do not add `2>/dev/null`, `|| echo`, `&&`, or any shell operators. If the directory or file does not exist, proceed normally to Step 0.

If a history file **exists**, read its front-matter to determine `mode`, `phase`, and `title`, then display a resume prompt:

```
Found an existing session — "<title>" [mode: <mode>].
Last completed phase: <phase> — <phase_name>

<brief summary of what was recorded in each completed phase>

Resume from Phase <phase+1>, or start over?
```

If the user chooses **resume**:
- Load all context from the history file (spec content, mode, architect plan, branch name, decisions, settings, etc.) — do not re-fetch or re-run what was already done.
- Jump to the next phase, reinjecting the saved context into any subagent prompts.

If the user chooses **start over**:

**[mode: issue only]** Steps 1–3 below apply only when the previous session was `mode: issue`:

1. Fetch all previous progress comments on the issue:

   ```bash
   gh issue view <number> --json comments --jq '.comments[] | select(.body | startswith("## [Phase")) | .databaseId'
   ```

2. Add a 👎 reaction to each comment ID:

   ```bash
   gh api repos/{owner}/{repo}/issues/comments/<comment_id>/reactions \
     --method POST \
     -f content="-1"
   ```

3. Post a reset announcement:

   ```bash
   gh issue comment <number> --body "## [Session Reset]

   A previous flow session for this issue has been discarded and a fresh session is starting. The prior phase comments above have been marked as voided (👎)."
   ```

4. Delete the existing history file.

5. Proceed normally from Step 0.

---

## History File

The history file is the source of truth for the state of a running session. It is stored in the **target repository** at:

- `mode: issue` → `.opencode/history/issue-<number>.md`
- `mode: text` → `.opencode/history/session-<slug>.md` (where `<slug>` is derived from the work title, lowercased, spaces → hyphens, max 40 chars)

### Format

```markdown
---
mode: <issue|text>
issue: <number or null>
title: "<work item title>"
branch: "<branch name or empty>"
phase: <last completed phase number>
phase_name: "<last completed phase name>"
last_updated: "<ISO 8601 timestamp>"
settings:
  base_branch: "<main|master|develop|...>"
  branch_format: "<pattern, e.g. feature/TD-{issue}-{slug}>"
  commit_format: "<pattern, e.g. feat: resolves #{issue} - {title}>"
  build_command: "<command or null>"
  lint_command: "<command or null>"
  test_command: "<command or null>"
---

## Phase 0 — Work Item Loaded
...

## Phase 1 — Triage & Scope
...

## Phase 2 — Analyze & Plan
...
```

### Usage tracking

At the end of each phase section, append a `### Usage` block:

```markdown
### Usage
- **Model:** <model-id>
- **Input tokens:** <count or "unknown">
- **Output tokens:** <count or "unknown">
```

Token counts come from the opencode UI at the end of each phase. If unavailable, write `unknown` — never fabricate numbers.

### Initialization

Before writing the first history entry, ensure `.opencode/history/` is excluded from git.

Use the **Read tool** (not bash) to read `.gitignore` and check whether it already contains `.opencode/` or `.opencode/history/`.

If not present, append it:

```bash
echo '.opencode/history/' >> .gitignore
```

Then create the directory:

```bash
mkdir -p .opencode/history
```

Run each bash command independently — never chain with `&&`, `||`, or `;`.

### Update cadence

Write or update the history file **at the end of each phase**, immediately before the pause point. Each write overwrites the entire file (update front-matter and append the new phase section).

---

## Step 0 — Load Work Item & Project Setup

### 0.1 — Determine input mode

Ask the user:

```
How do you want to specify the work to be done?

  1. GitHub issue — provide a number (#42) or URL
  2. Direct description — write or paste the specification here
```

**If the user chooses option 1 (mode: issue):**

Ask for the GitHub issue URL or short reference (`#<number>`). Accept:
- Full URL: `https://github.com/<owner>/<repo>/issues/<number>`
- Short reference: `#<number>`

Fetch the issue data:

```bash
gh issue view <number> --json number,title,body,labels,comments,assignees,state
```

Display a summary:

```
Issue #<number>: <title>
Labels: <labels>
State: <state>

<body>
```

Set `mode: issue`, `issue: <number>`, `title: <issue title>`.

**If the user chooses option 2 (mode: text):**

Ask the user to write or paste the specification:

```
Please describe the work to be done. Include as much detail as you have:
title, context, requirements, acceptance criteria, constraints — anything relevant.
```

Wait for the user's input. From that input, extract:
- **Title:** a short (max 10 words) title summarising the work — ask the user to confirm or adjust it
- **Description:** the full text as provided
- **Acceptance criteria:** if explicitly stated in the text, extract them; otherwise they will be derived by the triage subagent

Set `mode: text`, `issue: null`, `title: <extracted title>`.

### 0.2 — Sync with base branch

Ask the user:

```
¿Quieres hacer checkout a main y sincronizar (git pull) antes de empezar?
Current branch: <output of git branch --show-current>
```

```bash
git branch --show-current
```

If the user says yes:

```bash
git checkout main
```

```bash
git pull
```

Run each command independently. If `main` does not exist, try `master`. If neither exists, list available branches and ask the user which to use as the base.

Record the chosen base branch as `base_branch` in the settings (see 0.4).

### 0.3 — Check AGENTS.md

Use the **Read tool** (not bash) to read `AGENTS.md` in the target repository root.

- If the file **does not exist** or is **empty**: inform the user and offer to create a minimal one:

  ```
  This project does not have an AGENTS.md file. This file helps AI agents understand
  the project's conventions (commands, structure, coding standards).

  Would you like me to create a minimal AGENTS.md for this project?
  ```

  If the user agrees, create `AGENTS.md` with this template (adapt placeholders using what you already know):

  ```markdown
  # AGENTS.md

  ## Build / Lint / Test Commands

  | Command | Purpose |
  |---------|---------|
  | `<build command>` | Build the project |
  | `<lint command>` | Lint the code |
  | `<test command>` | Run the test suite |

  ## Project Structure

  <brief description of main directories>

  ## Conventions

  - Branch naming: <pattern>
  - Commit format: <pattern>
  ```

- If the file **exists and has content**: read it and load its conventions. Pay special attention to: build/lint/test commands, branch naming, commit format, and any project-specific rules.

### 0.4 — Load or build project settings

Project settings are stored in `.opencode/settings.json` within the **target repository**.

**Step 1 — Try to load existing settings:**

Use the **Read tool** to read `.opencode/settings.json`. If it exists and contains values, load them and skip the corresponding questions below.

**Step 2 — Infer missing settings from the project:**

For any setting not in `settings.json`, search (using the **Read tool**, in order, stop at first match):

- `CONTRIBUTING.md`
- `CODE-CONTRIBUTING.md`
- `CONTRIBUTING`
- `.github/CONTRIBUTING.md`
- `docs/CONTRIBUTING.md`

Also use `AGENTS.md` (already read in 0.3) and `README.md`.

Infer:
- `branch_format` (e.g. `feature/TD-{issue}-{slug}`, `issue-{issue}-{slug}`)
- `commit_format` (e.g. `feat: resolves #{issue} - {title}`, `[TD-{issue}] {title}`)
- `build_command`
- `lint_command`
- `test_command`

**Step 3 — Confirm with the user:**

```
Project settings detected:

- Base branch:    <value or "not set">
- Branch format:  <value or "not detected">
- Commit format:  <value or "not detected">
- Build command:  <value or "not detected">
- Lint command:   <value or "not detected">
- Test command:   <value or "not detected">

Are these correct? You can adjust any value before we continue.
```

Wait for the user to confirm or correct the values.

**Step 4 — Persist settings:**

```bash
mkdir -p .opencode
```

Write confirmed values to `.opencode/settings.json` using the **Write tool**:

```json
{
  "base_branch": "<value>",
  "branch_format": "<value or null>",
  "commit_format": "<value or null>",
  "build_command": "<value or null>",
  "lint_command": "<value or null>",
  "test_command": "<value or null>"
}
```

Ensure `.opencode/` is excluded from git (check `.gitignore` with the Read tool; append if needed).

### 0.5 — Confirm base branch for this work item

```
From which branch do you want to start the feature branch?
Current base branch: <base_branch from settings>

Available branches:
<output of git branch -a>

Press Enter to use <base_branch>, or type a different branch name:
```

```bash
git branch -a
```

Record the user's answer as the starting point for the branch created in Step 3.

### Write history — Phase 0

Initialize the history file:

- `mode: issue` → `.opencode/history/issue-<number>.md`
- `mode: text` → `.opencode/history/session-<slug>.md`

```markdown
---
mode: <issue|text>
issue: <number or null>
title: "<title>"
branch: ""
phase: 0
phase_name: "Work Item Loaded"
last_updated: "<ISO 8601 timestamp>"
settings:
  base_branch: "<value>"
  branch_format: "<value or null>"
  commit_format: "<value or null>"
  build_command: "<value or null>"
  lint_command: "<value or null>"
  test_command: "<value or null>"
---

## Phase 0 — Work Item Loaded
- **Mode:** <issue|text>
- **Issue:** <#number or "n/a — direct description">
- **Title:** <title>
- **Base branch:** <base_branch>

### Specification
<full issue body (mode: issue) or user-provided description (mode: text)>

### Comments  [mode: issue only — omit if mode: text]
<comments if any, or "No comments.">

### Usage
- **Model:** <model-id>
- **Input tokens:** <count or "unknown">
- **Output tokens:** <count or "unknown">
```

### Update issue — Phase 0 **[mode: issue only]**

```bash
gh issue edit <number> --add-assignee "@me"
```

```bash
gh issue comment <number> --body "## [Phase 0 — Work Item Loaded] ✓

Issue has been picked up and is now in active development.

**Next step:** Triage, requirements analysis, scope definition, and impact assessment."
```

**Pause and ask the user:** "Work item loaded. Shall I proceed with triage and scope analysis?"

---

## Step 1 — Triage & Scope

This phase characterises the work before any architectural analysis begins. It is executed by the **`triage` subagent**. The primary agent then presents the report to the user, handles clarifications, and (in `issue` mode) applies labels.

### 1.1 — Collect data and invoke the triage subagent

**[mode: issue only]** Refresh the issue data and fetch available labels:

```bash
gh issue view <number> --json number,title,body,labels,comments,assignees,state
gh label list --json name,description --limit 100
```

Invoke the `triage` subagent with the following context:

```
Work item to triage:

Mode: <issue|text>
<If mode: issue> Number: <number> / Title: <title> / State: <state> / Existing labels: <labels>
<If mode: text>  Title: <title>

Specification:
<full issue body (mode: issue) or user-provided description (mode: text)>

<If mode: issue>
Comments:
<comments if any, or "No comments.">

Repository labels available:  [include only if mode: issue]
<full output of gh label list>
```

The subagent will return a structured **Triage Report**. Do not proceed until it responds.

### 1.2 — Present the report and handle ambiguities

Display the full Triage Report to the user exactly as returned by the subagent.

If the report lists any **Ambiguities**, present them clearly and wait for the user to resolve them. Update the report with the user's answers before writing the history.

### 1.3 — Apply labels **[mode: issue only]**

```bash
gh label create "<type-label>" --color "<hex>" --description "<description>" 2>/dev/null || true
gh issue edit <number> --add-label "<type-label>"
```

Default colours: `documentation` → `#0075ca`, `enhancement` → `#a2eeef`, `bug` → `#d73a4a`, `fix` → `#e4e669`, `chore` → `#ededed`.

Remove any conflicting pre-existing type labels.

```bash
gh label create "in-analysis" --color "#fbca04" --description "Issue is being actively analysed" 2>/dev/null || true
gh issue edit <number> --add-label "in-analysis"
```

### 1.4 — Append acceptance criteria to the issue body **[mode: issue only]**

If the Triage Report indicates acceptance criteria were **derived** (not extracted from the issue):

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
- **Work item type:** <type>
- **Labels applied:** <list or "n/a — mode: text">

### Requirements
#### Functional
<list>

#### Non-functional
<list>

#### Acceptance criteria
- **Source:** <"Extracted from specification" or "Derived by triage agent" [+ "added to issue body" if mode:issue]>
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

### Update issue — Phase 1 **[mode: issue only]**

```bash
gh issue comment <number> --body "## [Phase 1 — Triage & Scope] ✓

**Type:** <type> (<label applied>)

**Acceptance criteria:** <'Extracted from specification' or 'Derived by triage agent and added to issue body'>

**Scope:**
<in-scope bullet points>

**Impact areas:**
<direct and indirect impact bullet points>

**Ambiguities to resolve:** <count or 'None'>

**Next step:** Architectural analysis and implementation planning."
```

**Pause and ask the user:** "Triage complete. Does the scope and impact analysis look correct? Shall I proceed with the architectural plan?"

---

## Step 2 — Analyze & Plan (architect subagent)

Invoke the `architect` subagent with the following context:

```
Analyze the following work item for this project and produce a complete implementation plan.

<If mode: issue>  Issue #<number>: <title>
<If mode: text>   Title: <title>

Specification:
<full body / user description>

<If mode: issue and comments exist>
Comments:
<comments>

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

### Update issue — Phase 2 **[mode: issue only]**

Post the **full architect analysis** as a comment on the issue:

```bash
gh issue comment <number> --body "## [Phase 2 — Analyze & Plan] ✓

### Analysis

<analysis section from the architect's output>

### Proposal

<proposal section from the architect's output>

### Implementation Plan

<implementation plan from the architect's output, step by step>

### Trade-offs

<trade-offs section from the architect's output>

---

*Plan reviewed and approved. Implementation starting next.*"
```

**Pause and ask the user:** "Does the plan look good? You can ask me to adjust it before we proceed."

---

## Step 3 — Branch, Implement & Build

### 3.1 — Choose the branch name

Load `branch_format` from `.opencode/settings.json`.

```bash
git branch -a
```

Generate a proposed branch name by applying `branch_format`:
- `{slug}` → work title lowercased, spaces → hyphens, special chars removed (max 40 chars)
- `{issue}` → issue number **if `mode: issue`**

**If `mode: text` and `branch_format` contains `{issue}`:** ask the user what identifier to use in place of `{issue}`:

```
The branch format contains {issue} but no GitHub issue is linked.
What identifier should I use instead? (e.g. a Jira ticket number, a date YYYYMMDD, a short ref, or just press Enter to omit it)
```

Use the user's answer to fill `{issue}`. If the user presses Enter (omit), remove the `{issue}-` fragment from the format.

If `branch_format` is null, propose `issue-{issue}-{slug}` (mode: issue) or `task-{slug}` (mode: text) as fallback.

Confirm with the user:

```
Proposed branch name: <proposed-name>

Existing branches (for reference):
<git branch -a output>

Press Enter to accept, or type a different name:
```

### 3.2 — Create the branch

```bash
git checkout -b <branch-name>
```

### 3.3 — Implement

Invoke the `general` subagent with the approved plan:

```
Implement the following plan. Load any relevant skills before starting.

<approved plan from architect>

Important:
- Follow all conventions in AGENTS.md.
- Load the relevant skills before writing code.
- Do not commit changes yet.
```

After the subagent finishes, summarize the files changed.

### 3.4 — Build the project

After implementation, verify the project compiles/builds successfully.

**Detect the build command** (first match wins):

1. `build_command` from `.opencode/settings.json`
2. **Read tool** → `Makefile`: look for a `build` target
3. **Read tool** → `README.md`: look for a build/compile/install section
4. Not found → ask the user:

   ```
   I could not detect the build command for this project.
   What command should I run to build/compile it? (e.g. `make build`, `npm run build`, `./gradlew build`)
   Type "skip" to skip the build step.
   ```

   Save the user's answer to `.opencode/settings.json` under `build_command`.

Run the build command independently. If it **fails**: show errors, fix automatically (invoke `general` again if needed), re-run until it passes. Do not ask for approval between iterations.

```
Build: PASSED
```

### Write history — Phase 3

Update the history file: set `phase: 3`, `phase_name: "Implementation"`, `branch: <branch-name>`, update `last_updated`, and append:

```markdown
## Phase 3 — Implementation
- **Branch:** <branch-name>
- **Files changed:**
<list each file, one per line, prefixed with "  -">

### Build
- **Command:** <build_command or "skipped">
- **Result:** PASSED (or SKIPPED)
- **Fix iterations required:** <number, or "0 — passed on first run">

### Summary
<summary as reported by the general subagent>

### Usage
- **Model:** <model-id>
- **Input tokens:** <count or "unknown">
- **Output tokens:** <count or "unknown">
```

### Update issue — Phase 3 **[mode: issue only]**

```bash
gh issue comment <number> --body "## [Phase 3 — Implementation] ✓

Implementation is complete on branch \`<branch-name>\`.

**Files changed:**
<list each file, one per line, prefixed with '- '>

**Build:** PASSED (or SKIPPED)

**Next step:** Running lint and tests."
```

**Pause and ask the user:** "Implementation complete. Review the changes above. Shall I proceed to run tests and linting?"

---

## Step 4 — Test & Lint

### 4.1 — Detect lint and test commands

Load `lint_command` and `test_command` from `.opencode/settings.json`.

For any command that is `null`, detect it now:

**Detection order (stop at first match):**

1. **Read tool** → `Makefile`: look for `lint`, `test`, `check` targets
2. **Read tool** → `README.md`: look for sections on running tests or linting
3. Not found → ask the user:

   ```
   I could not detect the <lint|test> command for this project.
   What command should I run? (e.g. `make lint`, `npm test`, `./gradlew test`)
   Type "skip" to skip this check.
   ```

   Save the answer to `.opencode/settings.json`.

### 4.2 — Run lint

```bash
<lint_command>
```

If it fails: show errors, fix automatically (invoke `general` if needed), re-run. No approval between iterations.

### 4.3 — Run tests

```bash
<test_command>
```

If it fails: show failing tests, fix automatically, re-run. Track fix iterations.

```
Lint: PASSED
Tests: PASSED
```

### Write history — Phase 4

Update the history file: set `phase: 4`, `phase_name: "Tests & Lint"`, update `last_updated`, and append:

```markdown
## Phase 4 — Tests & Lint
- **Lint command:** <lint_command or "skipped">
- **Lint:** PASSED (or SKIPPED)
- **Test command:** <test_command or "skipped">
- **Tests:** PASSED (or SKIPPED)
- **Fix iterations required:** <number, or "0 — passed on first run">

<If fix iterations > 0, briefly describe what was fixed>

### Usage
- **Model:** <model-id>
- **Input tokens:** <count or "unknown">
- **Output tokens:** <count or "unknown">
```

### Update issue — Phase 4 **[mode: issue only]**

```bash
gh issue comment <number> --body "## [Phase 4 — Tests & Lint] ✓

All quality checks have passed.

- **Lint:** PASSED (or SKIPPED)
- **Tests:** PASSED (or SKIPPED)
- **Fix iterations required:** <number, or '0 — passed on first run'>

**Next step:** Committing changes and opening a Pull Request."
```

**Pause and ask the user:** "All checks passed. Shall I commit the changes and open a Pull Request?"

---

## Step 5 — Commit & Pull Request

### 5.1 — Propose the commit message

Load `commit_format` from `.opencode/settings.json`.

Generate a proposed commit message:
- `{type}` → Conventional Commit type (`feat`, `fix`, `refactor`, `test`, `docs`, `chore`)
- `{title}` → work title (trimmed)
- `{slug}` → branch slug already used
- `{issue}` → issue number **if `mode: issue`**; if `mode: text` and format contains `{issue}`, use the same identifier chosen for the branch in Step 3.1, or omit it if the user chose to omit it

If `commit_format` is null, propose:
- `mode: issue` → `<type>: resolves #<number> - <title>`
- `mode: text` → `<type>: <title>`

Ask the user to confirm or adjust:

```
Proposed commit message:

  <proposed-commit-message>

Press Enter to accept, or type a different message:
```

If the user provides a new format pattern, ask:

```
Do you want to save this as the default commit format for this project?
(This will update .opencode/settings.json so you won't be asked again.)
```

If yes, update `commit_format` in `.opencode/settings.json`.

### 5.2 — Stage and commit

```bash
git add .
```

```bash
git commit -m "<confirmed commit message>"
```

### 5.3 — Open the Pull Request

**mode: issue:**

```bash
gh pr create \
  --title "<confirmed commit message>" \
  --body "$(cat <<'EOF'
## Summary

<1-3 bullet points describing what was changed and why>

## Changes

<list of key files changed>

## Testing

- [ ] Lint passes
- [ ] Tests pass

Closes #<number>
EOF
)"
```

**mode: text:**

```bash
gh pr create \
  --title "<confirmed commit message>" \
  --body "$(cat <<'EOF'
## Summary

<1-3 bullet points describing what was changed and why>

## Changes

<list of key files changed>

## Testing

- [ ] Lint passes
- [ ] Tests pass
EOF
)"
```

### Write history — Phase 5

Update the history file: set `phase: 5`, `phase_name: "Commit & PR"`, update `last_updated`, and append:

```markdown
## Phase 5 — Commit & PR
- **Commit message:** `<commit message>`
- **PR URL:** <url>

### Usage
- **Model:** <model-id>
- **Input tokens:** <count or "unknown">
- **Output tokens:** <count or "unknown">
```

### Update issue — Phase 5 **[mode: issue only]**

```bash
gh issue comment <number> --body "## [Phase 5 — Pull Request Opened] ✓

A Pull Request has been opened for review.

- **PR:** <pr-url>
- **Commit:** \`<commit message>\`

The issue will be closed automatically when the PR is merged."
```

### Post history as PR comment

After the PR is created, post the full history as a PR comment (always, both modes):

```bash
gh pr comment <pr-number> --body "$(cat <<'EOF'
<details>
<summary>Issue Flow History — <title></summary>

<full content of the history file, excluding the YAML front-matter block>

</details>
EOF
)"
```

Display the PR URL to the user.

---

## Conventions Reference

| Convention | Detail |
|---|---|
| Input mode | `issue` (GitHub issue) or `text` (direct description) — determined at Step 0.1 |
| History file | `issue: issue` → `.opencode/history/issue-<number>.md` / `mode: text` → `.opencode/history/session-<slug>.md` |
| Branch name | From `branch_format` in settings; `{issue}` asked to user if `mode: text` and format requires it |
| Commit format | From `commit_format` in settings; `{issue}` omitted or replaced if `mode: text` |
| PR body | Always includes Summary + Changes + Testing; `Closes #<number>` only if `mode: issue` |
| Build command | From settings; detected from Makefile/README or asked |
| Lint command | From settings; detected from Makefile/README or asked |
| Test command | From settings; detected from Makefile/README or asked |
| Settings file | `.opencode/settings.json` in the target repo (not version-controlled) |
| GitHub updates | Labels, comments, assignment — **only if `mode: issue`** |
| Null UUID for tests | `00000000-0000-0000-0000-000000000000` |
| History in git | Excluded via `.gitignore` — never committed |

## Skills to load during implementation

The `general` subagent should check available skills and load those relevant to the work item:

| Skill | When to load |
|---|---|
| `implement-api-resource` | Work involves creating a new API resource/endpoint |
| `test-api-resource` | Work involves writing or fixing tests for an API resource |
| `resolve-issue` | Always load this skill at the start of the flow |
