---
name: stale-worktrees
description: "Scan all git worktrees for completed tasks and report which ones can be removed. USE FOR: identifying stale worktrees whose linked tasks are in a terminal state (Done, Completed, In PR, Waiting Deployment). Read-only — never deletes worktrees."
---

# Stale Worktrees

Scan all git worktrees in the current repository, check each one for a `.task/task.json`, look up the linked work item, and report which worktrees are safe to remove because their task has reached a terminal state.

**This skill is strictly read-only. It MUST NOT delete, prune, or modify any worktree.**

## When to Use

* Periodic housekeeping — finding worktrees that are no longer needed
* Before running out of disk space or cleaning up branches
* After a batch of tasks has been merged / deployed

## Procedure

### 1. List All Worktrees

Run:

```sh
git worktree list --porcelain
```

Parse the output to extract the **path** of every worktree (including the main working tree).

### 2. Filter to Task Worktrees

For each worktree path, check whether a `.task` folder exists at its root.

* If `.task` does **not** exist → this is a custom / non-task worktree. **Skip it silently** — do not include it in any output.
* If `.task` exists → proceed to Step 3.

### 3. Read `task.json`

Look for `.task/task.json` inside the worktree root.

* If `task.json` is missing → note the worktree as **"task folder present but no task.json"** and skip further lookup.
* If `task.json` exists → read it and extract:
  * `taskLink` — the URL to the external work item
  * `taskSource` — the MCP namespace used to create the task (e.g. `mcp_makenotion`, `mcp_ado_wit`)
  * `slug`
  * `branch`
  * `description`

### 4. Look Up Work-Item Status (Subagent)

For each worktree with a valid `task.json`, launch a **subagent** to resolve the current status of the linked work item:

1. Determine the appropriate MCP from `taskSource` (fall back to inferring from the URL in `taskLink` if `taskSource` is absent).
2. Use `tool_search_tool_regex` to discover the read/get tool in that MCP namespace.
3. Call the tool to fetch the work item and extract its **status** (or equivalent field).

**MCP hints:**

| `taskSource` / URL pattern | Tool pattern to search |
|---|---|
| `mcp_makenotion` / `notion.so` | `mcp_makenotion` |
| `mcp_ado_wit` / `dev.azure.com` | `mcp_ado_wit` |

If the MCP call fails or the link is unreachable, mark the worktree as **"status unknown (lookup failed)"**.

### 5. Classify Worktrees

A worktree is **stale** if its work-item status matches any of these terminal states (case-insensitive):

* Done
* Completed
* Complete
* In PR
* Waiting Deployment
* Merged
* Closed
* Resolved
* Shipped

All other statuses are considered **active**.

### 6. Report

Present the results in a clear table or list grouped into two sections:

**Stale worktrees (safe to remove):**

For each stale worktree, show:

* Worktree path
* Branch name
* Task slug
* Task description (truncated to ~80 chars)
* Work-item status
* Link to work item

**Active worktrees (keep):**

For each active worktree, show the same fields so the user has full visibility.

If any worktrees had lookup failures, list them separately under **"Could not determine status"**.

End with a reminder: *"These are candidates for removal. To actually remove a worktree, run `git worktree remove <path>`."*

## Constraints

* **READ-ONLY** — never run `git worktree remove`, `git branch -d`, `rm -rf`, or any destructive command.
* Skip worktrees that have no `.task` folder — do not mention them in the report.
* Use subagents for parallel work-item lookups when there are multiple worktrees to check.
