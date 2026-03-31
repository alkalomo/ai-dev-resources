---
name: code-review
description: "Perform a senior-engineer code review on local changes. USE FOR: reviewing PRs, reviewing uncommitted changes, comparing branch against origin/main. Covers code quality, test coverage, test quality, permissions, cleanup, SOLID principles, and flags things that should not be checked in."
argument-hint: "Optional: path to specific files or folders to review, or 'staged' / 'unstaged' to limit scope"
---

# Code Review

Perform a thorough code review as a senior engineer on the current set of changes.

## When to Use

- Before creating a pull request
- After implementing a feature or bug fix
- When asked to review code changes
- As a self-review gate before committing

## Procedure

### 1. Gather the Diff

Determine which changes to review, in priority order:

1. **If the user provided specific files or folders as context** — review only those.
2. **If the user specified `staged` or `unstaged`** — use `get_changed_files` with that filter.
3. **If there are any staged or unstaged local changes** — use `get_changed_files` to retrieve them.
4. **Otherwise** — fetch `origin/main` and diff the current branch against it:
   ```
   git fetch origin main
   git diff origin/main...HEAD
   ```

If the diff is large, split the review into logical chunks (by directory or feature area) and review each chunk separately.

### 2. Understand Context

For each changed file:
- Read enough surrounding code to understand the intent of the change.
- Identify the feature, bug fix, or refactor being implemented.
- Note the project conventions from workspace instructions.

### 3. Review Checklist

Evaluate every change against **all seven areas** below. For each area, provide specific, actionable feedback referencing file names and line numbers.

#### 3.1 Code Quality
- Are the right language constructs, patterns, and framework APIs being used?
- Is existing shared code being reused rather than duplicated?
- Are naming conventions consistent with the rest of the codebase?
- Is the code readable and maintainable?

#### 3.2 Test Coverage
- Are there tests for the new or changed behavior?
- Are edge cases covered?
- Are negative/error paths tested?
- If tests are missing, identify exactly what should be tested.

#### 3.3 Test Quality
- Are the tests meaningful — do they assert the right behavior, not just implementation details?
- Are test names descriptive of the scenario being verified?
- Do tests follow Arrange-Act-Assert (or equivalent) structure?
- Are mocks/stubs appropriate, or are they hiding real bugs?

#### 3.4 Permission Management
- Are actions guarded by the correct permissions/authorization checks?
- Should any new endpoints or operations require permission checks that are missing?
- Are permission checks consistent with the existing patterns in the codebase?

#### 3.5 Cleanup & Dead Code
- If behavior changed or was replaced, was the old code removed?
- Are there unused imports, variables, methods, or classes left behind?
- Are configuration entries, feature flags, or DB migrations cleaned up?

#### 3.6 SOLID Principles
- **S** — Does each class/module have a single responsibility?
- **O** — Are changes open for extension without modifying existing code?
- **L** — Are subtypes substitutable for their base types?
- **I** — Are interfaces lean and specific rather than bloated?
- **D** — Are dependencies injected, not hard-coded?

#### 3.7 Should Not Be Checked In
Flag any of the following:
- Hardcoded secrets, tokens, or credentials
- Debug/console logging left in production code
- Commented-out code blocks
- TODO/HACK comments without tracking references
- Temporary workarounds or test scaffolding
- Files that appear unrelated to the change (accidental staging)

### 4. Produce the Review

Output the review in this format:

```
## Code Review Summary

**Branch**: <branch name>
**Files reviewed**: <count>
**Overall assessment**: 🟢 Approve / 🟡 Approve with comments / 🔴 Request changes

### Critical Issues
<issues that must be fixed before merge>

### Suggestions
<improvements that are recommended but not blocking>

### Nits
<minor style or preference items>

### Positive Callouts
<things done well worth highlighting>
```

- If there are **no issues** in a review area, skip it — do not add filler.
- Be specific: reference file names and line numbers for every finding.
- For each issue, briefly explain **why** it matters, not just what to change.
