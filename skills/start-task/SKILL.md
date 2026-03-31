---
name: start-task
description: "Create an isolated worktree, bootstrap dependencies, and create task.json. USE FOR: starting new feature or bug-fix work before planning begins."
argument-hint: "Describe the task or paste a link to a task (e.g. Notion, ADO, etc.)"
---

# Start Task

Create and bootstrap a new git worktree for a task, then write a `task.json` file with the problem statement and metadata.

## When to Use

* Starting a new feature or bug fix
* Need an isolated working copy with dependencies installed
* Want to capture the problem statement before planning begins

## Inputs

The user provides **one** of the following:

* **Task description**: A brief text description of the task.
* **Task link**: A URL pointing to a task in an external tool (e.g. Notion page, ADO work item, GitHub issue). The link will be resolved via the appropriate MCP in Step 0.

## Procedure

### 0. Resolve Task Input

Determine whether the user provided a **plain-text description** or a **link**.

**If the input is a link / URL:**

1. Identify which MCP can handle the URL (e.g. Notion MCP for Notion links, ADO MCP for Azure DevOps links).
2. Use `tool_search_tool_regex` to discover the appropriate MCP read tool, then call it to fetch the task details (title, description, acceptance criteria, etc.).
3. Compose a consolidated **task description** from the fetched data. This description is used for all subsequent steps (slug generation, task.json, etc.).
4. Keep a reference to the original link and the MCP namespace used — both are persisted in `task.json`.

**If the input is plain text:** use it directly as the task description. No MCP interaction is needed.

### 1. Generate Slug

Derive a short, URL-safe slug from the task description:

* Lowercase, hyphen-separated, max 40 characters
* Remove filler words (the, a, an, for, to, of, in, on, with)
* Examples:

  * "Fix metrics tab crash on empty data" → `fix-metrics-tab-crash-empty-data`
  * "Add brand voice export feature" → `add-brand-voice-export`

### 2. Determine Scope

Decide whether the task targets a **specific project subfolder** or the **entire repository**.

1. Scan the worktree for project subfolders — directories that contain their own dependency manifests (`package.json`, `requirements.txt`, `pyproject.toml`, etc.). Check common monorepo nesting patterns: top-level directories and one level deep inside folders like `tools/`, `packages/`, `apps/`, `services/`, `projects/`, `libs/`.
2. Match the task description against discovered project names and paths. For example, "fix SemantIQ login bug" clearly maps to a subfolder named `SemantIQ`.
3. If exactly one project matches, set **scope = `"project"`** and record its relative path.
4. If multiple projects match or no project matches, ask the user: *"Is this task scoped to a specific project subfolder, or does it span the whole repo?"*
5. If the user confirms repo-wide, set **scope = `"repo"`**. For repo-scoped tasks, identify all project subfolders that are likely relevant to the task (based on the description) — these will be bootstrapped in Step 6.

**Result:** `scope` (`"project"` or `"repo"`) and `project` (relative path from worktree root, or `null` for repo-scoped).

### 3. Determine Paths

All paths are derived from the **git root** (use `git rev-parse --show-toplevel`):

* **Git root**: result of `git rev-parse --show-toplevel`
* **Worktree root**: `<git-root>/../worktrees/<slug>`
* **Branch name**: `<git-user>/<slug>` where `<git-user>` comes from the local part of `git config user.email` (before the `@`)
* **Task directory**: `<worktree-root>/.task`
* **Task file**: `<task-directory>/task.json`

The `.task` directory is always at the worktree root, regardless of scope.

### 4. Create the Worktree

Run:

```sh
git fetch origin main
git worktree add --no-track -b <branch-name> <worktree-root> origin/main
```

If the branch already exists, abort and inform the user.

### 5. Create the Task Directory

Ensure the task directory exists:

```sh
mkdir -p <task-directory>
```

If on Windows shells where `mkdir -p` is unavailable, use the shell-appropriate equivalent.

### 6. Bootstrap Dependencies (background)

For each project directory that needs bootstrapping, auto-detect the dependency manager from marker files and run the install in a **background terminal**. Each directory gets its own background terminal.

**Which directories to bootstrap:**

* **Project-scoped**: the single project directory (`<worktree-root>/<project>`)
* **Repo-scoped**: all project directories identified in Step 2 as relevant to the task. If the root `package.json` has a `"workspaces"` field, run a single `npm install` (or equivalent) at the worktree root instead of per-subfolder installs.

**Detection rules** (check in order, use the first match):

| Marker file | Command |
|-------------|---------|
| `package-lock.json` | `npm install` |
| `yarn.lock` | `yarn install` |
| `pnpm-lock.yaml` | `pnpm install` |
| `package.json` (no lockfile) | `npm install` |
| `requirements.txt` | Create venv + `pip install -r requirements.txt` (see below) |
| `pyproject.toml` | Create venv + `pip install -e .` (see below) |

**Python venv commands:**

* **Windows**: `py -m venv .venv && .venv\Scripts\Activate.ps1 && pip install ...`
* **Unix**: `python3 -m venv .venv && source .venv/bin/activate && pip install ...`

If no marker files are detected in a directory, skip it silently.

### 7. Create `.task/task.json`

Write a JSON file capturing the problem statement and metadata:

```json
{
  "slug": "<slug>",
  "branch": "<branch-name>",
  "worktreeRoot": "<absolute path to worktree root>",
  "scope": "project | repo",
  "project": "<relative path from worktree root> | null",
  "bootstrappedProjects": ["<relative-path>", "..."],
  "description": "<consolidated task description>",
  "taskLink": "<original URL or null>",
  "taskSource": "<MCP namespace used to resolve the link, or null>",
  "createdAt": "<ISO 8601 timestamp>"
}
```

Field details:

| Field | Type | Description |
|-------|------|-------------|
| `slug` | `string` | The URL-safe slug derived in Step 1 |
| `branch` | `string` | The git branch created in Step 4 |
| `worktreeRoot` | `string` | Absolute path to the worktree root |
| `scope` | `"project" \| "repo"` | Whether the task targets a single project subfolder or the full repo |
| `project` | `string \| null` | Relative path from worktree root to the project subfolder, or `null` for repo-scoped tasks |
| `bootstrappedProjects` | `string[]` | Relative paths of directories that were bootstrapped |
| `description` | `string` | The full task description (from user input or resolved from link) |
| `taskLink` | `string \| null` | The original task URL, or `null` if plain-text input was provided |
| `taskSource` | `string \| null` | The MCP namespace used to resolve the link (e.g. `mcp_ado_wit`, `mcp_makenotion`), or `null` |
| `createdAt` | `string` | ISO 8601 timestamp of when the task was created |

### 8. Update Task Status (if applicable)

If the task was resolved from a link in Step 0:

1. Use `tool_search_tool_regex` to discover an update/edit tool in the same MCP namespace identified in Step 0.
2. If an update tool exists, set the task status to **"Ready for Planning"** (or the closest equivalent the tool supports).
3. If the MCP does not support status updates, skip this step silently.

### 9. Summary

Report:

* Worktree path
* Branch name
* Scope (`project` with path, or `repo`)
* Task file path (`.task/task.json`)
* Which directories were bootstrapped (and which package manager was used for each)
* Whether the task status was updated (and to what), or that no update was needed
* That dependency installs are running in background terminals
