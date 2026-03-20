# AGENTS.md — opencode Dev Workflow

This is a **global opencode configuration repository**. It contains agent definitions
(`agents/`) and skill protocols (`skills/`) used by the opencode AI assistant across
target repositories. There is no application source code here.

---

## Repository Structure

```
agents/          # Agent definitions (Markdown + YAML front-matter)
skills/          # Reusable skill protocols loaded at runtime
  resolve-issue/
    SKILL.md
node_modules/    # bun-managed; not tracked in git
```

Key files:
- `agents/issue-flow.md` — Primary orchestrator agent for GitHub issue resolution
- `agents/architect.md` — Read-only architecture analysis subagent
- `agents/triage.md` — Hidden subagent for structured issue triage reports (no tool access)
- `skills/resolve-issue/SKILL.md` — Full protocol for end-to-end issue resolution

---

## Package Manager

This repo uses **bun**. The only direct dependency is `@opencode-ai/plugin`.

```bash
bun install     # Install/update dependencies
```

`package.json` and `bun.lock` are listed in `.gitignore` and not tracked in git.
Only `agents/` and `skills/` content is version-controlled.

---

## Build / Lint / Test Commands

This repo has **no build step, no linter, and no test suite of its own**.

When agents operate on *target* repositories, they use the target repo's commands:

| Command      | Purpose                            |
|--------------|------------------------------------|
| `make lint`  | Lint the target project            |
| `make test`  | Run the full test suite            |

To run a single test in a target repo, check that repo's `AGENTS.md` or `Makefile`
for the appropriate invocation (e.g., `make test TEST=path/to/test`).

---

## Agent File Format

Every agent file at `agents/<name>.md` must have a YAML front-matter block followed
by a Markdown system prompt.

### Required front-matter fields

```yaml
---
description: <one-line description shown in agent picker>
mode: primary | subagent
temperature: <float 0.0–1.0>
permission:
  edit: allow | deny
  bash:
    "*": deny           # Default-deny is strongly preferred
    "git log*": allow   # Then allowlist specific patterns
  webfetch: allow | deny
---
```

**Permission rules:**
- Always start with `"*": deny` under `bash` and only allowlist what is necessary.
- Read-only agents (like `architect`) must set `edit: deny`.
- Agents that create commits/PRs need `git add*`, `git commit*`, `gh pr*`, etc.
- Agents that manage labels need `gh label*` explicitly allowlisted.
- Pure-reasoning subagents (like `triage`) that receive all context via prompt should have `bash: {"*": deny}` and no allowlist entries.

### Mode semantics

| Mode       | Meaning                                                     |
|------------|-------------------------------------------------------------|
| `primary`  | User-facing; drives the top-level conversation flow         |
| `subagent` | Invoked by another agent; not directly accessible by users  |

---

## Skill File Format

Every skill lives at `skills/<name>/SKILL.md` with front-matter:

```yaml
---
name: <skill-name>         # Used in skill({ name: "..." }) calls
description: <one-liner>
---
```

The Markdown body is the skill's complete protocol, loaded at runtime when an agent
calls `skill({ name: "<skill-name>" })`.

**Naming convention:** skill directories use `lowercase-kebab-case`.
The file is always named `SKILL.md` (uppercase).

---

## Naming Conventions

| Entity              | Convention                                          | Example                                     |
|---------------------|-----------------------------------------------------|---------------------------------------------|
| Agent files         | `lowercase-kebab-case.md`                           | `issue-flow.md`, `architect.md`             |
| Skill directories   | `lowercase-kebab-case/`                             | `resolve-issue/`                            |
| Skill file          | `SKILL.md` (uppercase)                              | `skills/resolve-issue/SKILL.md`             |
| Git branches        | `issue-<number>-<slug>` (≤ 60 chars total)          | `issue-42-add-pagination-to-items-endpoint` |
| Commit messages     | Conventional Commits: `<type>: resolves #<n> - <title>` | `feat: resolves #42 - Add pagination`   |
| PR titles           | Same as commit message                              | —                                           |

### Conventional Commit types

| Type       | When to use                                        |
|------------|----------------------------------------------------|
| `feat`     | New feature or endpoint                            |
| `fix`      | Bug fix                                            |
| `refactor` | Code restructuring without behavior change         |
| `test`     | Test additions or fixes                            |
| `docs`     | Documentation only                                 |
| `chore`    | Tooling, dependencies, configuration               |

---

## Workflow Conventions (resolve-issue skill)

When using the `issue-flow` agent or `resolve-issue` skill, the following phases apply:

| Phase | Action                                                              | Pause point                                              |
|-------|---------------------------------------------------------------------|----------------------------------------------------------|
| -1    | Check for existing history file (resume detection)                  | "Resume from Phase N, or start over?"                    |
| 0     | Fetch issue with `gh issue view`; assign `@me`                      | "Shall I proceed with triage?"                           |
| 1     | Triage: classify type (label), analyse requirements, scope, impact  | "Does the scope and impact look correct?"                |
| 2     | Invoke `architect` subagent for a plan                              | "Does the plan look good?"                               |
| 3     | Create branch + invoke `general` subagent                           | "Shall I proceed to run tests?"                          |
| 4     | Run `make lint` then `make test`                                    | "Shall I commit and open a PR?"                          |
| 5     | `git add .`, `git commit`, `gh pr create`, post history comment     | Show PR URL                                              |

**Never skip a phase without explicit user approval.**

### Usage tracking

At the end of each phase, the agent records a `### Usage` block in the history file with:

```markdown
### Usage
- **Model:** <model-id>
- **Input tokens:** <count or "unknown">
- **Output tokens:** <count or "unknown">
```

Token counts are read from the opencode UI at the end of each phase. If unavailable, write `unknown` — never fabricate numbers. This allows per-phase cost analysis across a full issue resolution session.

### PR body template

```
## Summary
<1-3 bullet points describing what changed and why>

## Changes
<list of key files changed>

## Testing
- [ ] `make lint` passes
- [ ] `make test` passes

Closes #<number>
```

---

## Error Handling in Agents

- If `make lint` or `make test` fail: show errors, automatically fix and re-run.
  Do **not** ask for approval on each iteration — fix silently until all pass.
- If the user cancels mid-flow: leave the branch and uncommitted changes in place;
  inform the user of the current state.
- If the user wants to skip a phase: acknowledge and skip, but warn of any risk.
- If the user wants to change the plan mid-flow: update the plan and re-summarize
  before continuing.

---

## Test Fixtures Convention

- Use `00000000-0000-0000-0000-000000000000` as the null/placeholder UUID in tests.

---

## Adding a New Agent

1. Create `agents/<name>.md`.
2. Add YAML front-matter with `description`, `mode`, `temperature`, and `permission`.
3. Write the system prompt in the Markdown body.
4. Start `bash` permissions with `"*": deny` and allowlist only required patterns.
5. Commit with: `chore: add <name> agent`

## Adding a New Skill

1. Create `skills/<name>/SKILL.md`.
2. Add front-matter with `name` and `description`.
3. Write the full protocol in the Markdown body.
4. Document which agents should load the skill and under what conditions.
5. Commit with: `chore: add <name> skill`
