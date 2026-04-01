---
name: code-review
description: "Perform a senior-engineer code review on local or remote changes using sub-agents. USE FOR: reviewing PRs (local or remote), reviewing uncommitted changes, comparing branch against origin/main. Orchestrates sub-agents for diff collection, change analysis, and parallel review of code quality, tests, permissions, cleanup, SOLID, and hygiene."
argument-hint: "Optional: PR link, path to specific files/folders, or 'staged' / 'unstaged' to limit scope"
---

# Code Review — Orchestrator

You are the **orchestrator**. You launch sub-agents and present the final review. **Do not read source files, diffs, or review outputs yourself.** All heavy work — including aggregation — is delegated to sub-agents. Your only direct responsibilities are creating the workspace, launching sub-agents in sequence, and displaying the final summary.

## When to Use

- Before creating a pull request
- After implementing a feature or bug fix
- When asked to review code changes (local or remote PR)
- As a self-review gate before committing

## Architecture

Sub-agents communicate through files in a **shared temp folder**, not through the orchestrator's context. Each sub-agent writes its output to a well-known path; the next sub-agent reads from there.

```
User Input
    │
    ▼
┌──────────────────────┐
│  Phase 0: Setup       │  ← Orchestrator: create temp folder
│  (.code-review/)      │
└────────┬─────────────┘
         │
         ▼
┌──────────────────────┐
│  Phase 1: Collect     │  ← Sub-agent: Diff Collector
│  writes → diff.md     │
└────────┬─────────────┘
         │
         ▼
┌──────────────────────┐
│  Phase 2: Analyze     │  ← Sub-agent: Change Analyst
│  reads  ← diff.md     │
│  writes → analysis.md │
└────────┬─────────────┘
         │
         ▼
┌──────────────────────┐
│  Phase 3: Review      │  ← Sub-agents: Review Workers (parallel)
│  reads  ← analysis.md │     One per review group
│  writes → review-N.md │
└────────┬─────────────┘
         │
         ▼
┌──────────────────────┐
│  Phase 4: Aggregate   │  ← Sub-agent: Aggregator
│  reads  ← analysis.md │
│          + review-*.md │
│  writes → summary.md  │
└────────┬─────────────┘
         │
         ▼
┌──────────────────────┐
│  Phase 5: Present     │  ← Orchestrator: read summary.md
│  (display to user)    │     and display to user
└──────────────────────┘
```

### Temp Folder Layout

```
.code-review/
├── diff.md          # Phase 1 output — metadata + changed-file list
├── analysis.md      # Phase 2 output — change summary + review groups
├── review-1.md      # Phase 3 output — findings for group 1
├── review-2.md      #   "
├── review-N.md      #   "
└── summary.md       # Phase 4 output — aggregated final review
```

All paths below are relative to the workspace root. The orchestrator creates `.code-review/` at the start and each sub-agent writes to its designated file(s).

## Procedure

### Phase 0 — Setup (Orchestrator)

Create the temp folder for this review session:

```sh
mkdir -p .code-review
```

On Windows, use the shell-appropriate equivalent. If `.code-review/` already exists from a previous run, delete its contents first.

### Phase 1 — Diff Collection (Sub-agent)

Launch a sub-agent with the prompt below. Pass along any user-provided context (PR link, file paths, `staged`/`unstaged`).

> **Sub-agent prompt — Diff Collector**
>
> You are a diff-collection assistant. Your job is to obtain the set of changed files and write a structured summary to disk. Do NOT review the code — only collect.
>
> **Determine the source of changes (in priority order):**
>
> 1. **Remote PR link provided** — The user gave a PR URL.
>    - Identify the hosting platform from the URL (GitHub, Azure DevOps, etc.).
>    - Use `tool_search_tool_regex` to find the appropriate MCP tool to fetch PR details and diff (e.g. `mcp_ado_repo` for ADO, `mcp_gitkraken` or `github_repo` for GitHub).
>    - Fetch the PR metadata (title, description, source branch, target branch) and the full diff.
> 2. **User provided specific files or folders** — List only those paths and obtain their diffs via `git diff` or `get_changed_files`.
> 3. **User specified `staged` or `unstaged`** — Use `get_changed_files` with that filter.
> 4. **Local changes exist** — Use `get_changed_files` to detect staged and unstaged changes.
> 5. **No local changes** — Diff the current branch against `origin/main`:
>    ```
>    git fetch origin main
>    git diff origin/main...HEAD --stat
>    git diff origin/main...HEAD
>    ```
>
> **Write to `.code-review/diff.md`** with exactly this structure:
> ```
> METADATA:
>   source: <"pr" | "local-staged" | "local-unstaged" | "branch-diff">
>   branch: <branch name>
>   pr_link: <URL or "N/A">
>   pr_title: <title or "N/A">
>   pr_description: <description or "N/A">
>   target_branch: <target branch or "N/A">
>
> CHANGED FILES:
>   - <path> (<added/modified/deleted/renamed>, +<lines> -<lines>)
>   ...
>
> DIFF STATS:
>   total_files: <N>
>   total_additions: <N>
>   total_deletions: <N>
> ```
>
> Confirm that the file was written successfully. Return only: "Done. Wrote .code-review/diff.md with <N> changed files."

**After the sub-agent returns**, do NOT read `diff.md` — the next sub-agent will read it directly.

### Phase 2 — Change Analysis (Sub-agent)

Launch a sub-agent with the following prompt. Do NOT pass it the diff data — it reads from disk.

> **Sub-agent prompt — Change Analyst**
>
> You are a change-analysis assistant. Your job is to understand what this change is about and organize files into logical review groups. Do NOT review the code for issues — only analyze and categorize.
>
> **Input:** Read `.code-review/diff.md` for the changed-file list and metadata.
>
> **Your tasks:**
> 1. Read the changed files and enough surrounding code to understand the purpose of each change.
> 2. Identify the overall intent: feature, bug fix, refactor, config change, dependency update, etc.
> 3. Group the changed files into **logical review groups**. Each file must appear in exactly one group. Use your judgment on the right number of groups — this depends entirely on the shape of the change:
>    - A small, focused change (≤5 files) might warrant a single group.
>    - A large cross-cutting change might need up to 10 groups.
>    - **Group by logical cohesion**, not by rigid categories. A single critical file that was heavily modified can be its own group. A set of tightly coupled files that implement one behavior should be grouped together, even if they span layers.
>    - Do NOT force artificial groupings like "API layer" / "Tests" / "Config" — let the structure of the actual change drive the breakdown.
> 4. For each group, write a one-sentence summary of what changed in that area.
>
> **Write to `.code-review/analysis.md`** with exactly this structure:
> ```
> CHANGE SUMMARY:
>   intent: <one-sentence description of what this change accomplishes>
>   type: <feature | bugfix | refactor | config | dependency | mixed>
>
> REVIEW GROUPS:
>   - group: "<group name>"
>     id: 1
>     summary: "<what changed in this area>"
>     files:
>       - <path>
>       - <path>
>
>   - group: "<group name>"
>     id: 2
>     summary: "<what changed in this area>"
>     files:
>       - <path>
>       ...
> ```
>
> Confirm that the file was written successfully. Return only: "Done. Wrote .code-review/analysis.md with <N> review groups."

**After the sub-agent returns**, do NOT read `analysis.md`. Read only the brief confirmation to know how many groups were created — you need this count to launch the correct number of Phase 3 workers.

### Phase 3 — Review Workers (Sub-agents, parallel)

Launch **one sub-agent per review group**. All workers run in parallel. Each worker reads its assignment from `analysis.md`, reviews its files, and writes findings to `review-{id}.md`.

> **Sub-agent prompt — Review Worker #{GROUP_ID}**
>
> You are a senior-engineer code reviewer. Your job is to review one group of files and write your findings to disk.
>
> **Setup:**
> 1. Read `.code-review/analysis.md`.
> 2. Find the group with `id: {GROUP_ID}`. That is your assignment — note the group name, summary, and file list.
> 3. Note the overall `CHANGE SUMMARY` (intent and type) for context.
>
> **Review instructions:**
> 1. Read each file in your group in full, plus enough surrounding code to understand context.
> 2. Evaluate against **all seven review areas** below. Be specific — reference file paths and line numbers.
> 3. If a review area has no findings for your files, omit it entirely.
>
> **Review Checklist:**
>
> **1. Code Quality**
> - Are the right language constructs, patterns, and framework APIs being used?
> - Is existing shared code being reused rather than duplicated?
> - Are naming conventions consistent with the rest of the codebase?
> - Is the code readable and maintainable?
>
> **2. Test Coverage**
> - Are there tests for the new or changed behavior?
> - Are edge cases covered?
> - Are negative/error paths tested?
> - If tests are missing, identify exactly what should be tested.
>
> **3. Test Quality**
> - Are the tests meaningful — do they assert the right behavior, not just implementation details?
> - Are test names descriptive of the scenario being verified?
> - Do tests follow Arrange-Act-Assert (or equivalent) structure?
> - Are mocks/stubs appropriate, or are they hiding real bugs?
>
> **4. Permission Management**
> - Are actions guarded by the correct permissions/authorization checks?
> - Should any new endpoints or operations require permission checks that are missing?
> - Are permission checks consistent with the existing patterns in the codebase?
>
> **5. Cleanup & Dead Code**
> - If behavior changed or was replaced, was the old code removed?
> - Are there unused imports, variables, methods, or classes left behind?
> - Are configuration entries, feature flags, or DB migrations cleaned up?
>
> **6. SOLID Principles**
> - **S** — Does each class/module have a single responsibility?
> - **O** — Are changes open for extension without modifying existing code?
> - **L** — Are subtypes substitutable for their base types?
> - **I** — Are interfaces lean and specific rather than bloated?
> - **D** — Are dependencies injected, not hard-coded?
>
> **7. Should Not Be Checked In**
> - Hardcoded secrets, tokens, or credentials
> - Debug/console logging left in production code
> - Commented-out code blocks
> - TODO/HACK comments without tracking references
> - Temporary workarounds or test scaffolding
> - Files that appear unrelated to the change (accidental staging)
>
> **Write to `.code-review/review-{GROUP_ID}.md`** with exactly this structure:
> ```
> GROUP: <group name>
> FILES REVIEWED: <count>
>
> CRITICAL:
>   - [<file>:<line>] <description of issue and why it matters>
>   ...
>
> SUGGESTIONS:
>   - [<file>:<line>] <description and rationale>
>   ...
>
> NITS:
>   - [<file>:<line>] <description>
>   ...
>
> POSITIVE:
>   - [<file>:<line>] <what was done well>
>   ...
> ```
> Omit any section (CRITICAL, SUGGESTIONS, NITS, POSITIVE) that has no items.
>
> Confirm that the file was written successfully. Return only: "Done. Wrote .code-review/review-{GROUP_ID}.md."

**After all workers return**, do NOT read the review files yourself. Proceed to Phase 4.

### Phase 4 — Aggregation (Sub-agent)

Launch a sub-agent to read all review outputs and the change analysis, then produce the final consolidated summary.

> **Sub-agent prompt — Aggregator**
>
> You are a review-aggregation assistant. Your job is to read the change analysis and all review findings, then produce a single consolidated code review summary.
>
> **Input files:**
> 1. Read `.code-review/analysis.md` for the change summary, intent, and review group names.
> 2. Read all `.code-review/review-*.md` files for the findings from each review group.
>
> **Aggregation rules:**
> 1. **Determine overall assessment:**
>    - Any CRITICAL findings → 🔴 Request changes
>    - Only SUGGESTIONS or NITS → 🟡 Approve with comments
>    - No findings (or only POSITIVE) → 🟢 Approve
> 2. **Deduplicate** findings that appear in multiple groups (keep the most specific one).
> 3. **Sort** CRITICAL issues first by severity, then by file path.
> 4. For each finding, preserve the file path and line number from the worker.
> 5. Briefly explain **why** each issue matters, not just what to change.
> 6. Tag each finding with its source group name for traceability.
> 7. Omit any section that has no items — do not add filler.
>
> Also read `.code-review/diff.md` for the branch name, PR link, and file count.
>
> **Write to `.code-review/summary.md`** with exactly this structure:
> ```
> ## Code Review Summary
>
> **Branch**: <branch name>
> **PR**: <link or "local changes">
> **Change**: <one-sentence intent from analysis.md>
> **Files reviewed**: <total count>
> **Overall assessment**: 🟢 Approve / 🟡 Approve with comments / 🔴 Request changes
>
> ### Critical Issues
> - [file:line] description *(from: group name)*
> ...
>
> ### Suggestions
> - [file:line] description *(from: group name)*
> ...
>
> ### Nits
> - [file:line] description *(from: group name)*
> ...
>
> ### Positive Callouts
> - [file:line] description *(from: group name)*
> ...
> ```
>
> Confirm that the file was written successfully. Return only: "Done. Wrote .code-review/summary.md — assessment: <🟢|🟡|🔴>."

### Phase 5 — Present (Orchestrator)

Read `.code-review/summary.md` and display its contents to the user verbatim. This is the only file the orchestrator reads during the entire review.
