---
name: plan-task
description: "Generate an implementation plan for a task. USE FOR: creating a reviewed implementation plan after start-task has set up the environment. Reads .task/task.json for context."
argument-hint: "No arguments needed — reads from .task/task.json"
---

# Plan Task

Analyze the codebase and write a Markdown implementation plan to disk. Reads task context from `.task/task.json` (created by `start-task`).

## When to Use

* After `start-task` has created the worktree and bootstrapped dependencies
* Ready to generate a durable implementation plan before making code changes

## Prerequisites

The `start-task` skill must have been run first. It creates `.task/task.json` which this skill reads for the task description, slug, and metadata.

## Procedure

### 0. Locate and Read `.task/task.json`

Look for `.task/task.json` relative to the current workspace root.

**If the file is missing:** stop and tell the user to run `start-task` first.

Parse the JSON and extract:

| Field | Used for |
|-------|----------|
| `slug` | Context only (plan file path is fixed) |
| `description` | Understanding the task to plan for |
| `scope` | Whether to focus on a single project or the full repo |
| `project` | The specific project subfolder to focus on (if project-scoped) |
| `bootstrappedProjects` | Hint for which areas of the repo are involved |
| `taskLink` | Metadata comment in the plan file; status updates |
| `taskSource` | Identifying the MCP namespace for status updates |

### 1. Analyze the Codebase

Using the `description` from `task.json`, research the codebase and plan how to address the task.

* **Project-scoped** (`scope` = `"project"`): Focus analysis on the directory identified in `project`. Start by exploring that subtree; only look outside it if the task requires cross-project changes.
* **Repo-scoped** (`scope` = `"repo"`): Analyze across the full repo. Use `bootstrappedProjects` as a starting hint for which areas are most likely involved, but do not limit analysis to only those directories.

### 2. Write the Plan

Write the implementation plan to `.task/plan.md`.

### 3. Update Task Status (if applicable)

If `taskLink` and `taskSource` are present in `task.json`:

1. Use `tool_search_tool_regex` to discover an update/edit tool in the MCP namespace stored in `taskSource`.
2. If an update tool exists, set the task status to **"Plan Ready"** (or the closest equivalent the tool supports).
3. If the MCP does not support status updates, skip this step silently.

### 4. Summary

Report:

* Plan file path (`.task/plan.md`)
* Whether the task status was updated (and to what), or that no update was needed
* Any issues encountered
