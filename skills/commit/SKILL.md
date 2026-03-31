---
name: commit
description: "Looks at the staged changes, creates a commit message and commits the changes. Optionally pushes the commit to the remote branch. USE FOR: committing code changes after implementing a task, with an informative commit message."
argument-hint: "Optional argument --push to also push the commit to the remote branch"
---

Look at the staged changes, create a commit message, and commit the changes. If the `--push` argument is provided, also push the commit to the remote branch.

In case the branch is not already tracked, publish the branch to the remote while pushing.