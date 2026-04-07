---
name: plugin-status
description: Audit the current state of a WordPress plugin against its task list. Reads the tasks file and inspects the actual codebase to report what's done, what's incomplete, and what's missing entirely.
disable-model-invocation: true
argument-hint: [path-to-tasks-file]
allowed-tools: Read, Bash, Grep, Glob
---

# Plugin Status Report

You are auditing a WordPress plugin's current state against its task list. You will read the task file, then inspect the actual code to verify whether each task has genuinely been completed — not just whether someone ticked a checkbox.

## Input

Task file to use: **$ARGUMENTS**

If no argument is provided, look for `TASKS.md` in the project root. If multiple `TASKS-*.md` files exist, list them and ask the user which one to audit.

## Process

### Step 1: Load the task list

Read the full task file. Build a list of every task with its number, title, description, expected test, and current checkbox state (`[x]` or `[ ]`).

### Step 2: Inspect the codebase

For each task, verify whether the work described actually exists in the code. Don't trust the checkbox — verify against the source. Techniques to use:

- **File existence**: Does the file the task references exist? (`Glob`, `Bash(ls)`)
- **Function/class existence**: Does the function, class, hook registration, or route actually exist? (`Grep` for function names, class names, `add_action`, `register_post_type`, `register_rest_route`, etc.)
- **Implementation completeness**: Is it a stub or a real implementation? Read the file and check for placeholder comments like `// TODO`, empty method bodies, or missing logic.
- **DocBlocks**: Does the function have a proper DocBlock with `@param` and `@return`?
- **Security**: For tasks involving forms, AJAX, or REST endpoints — is there a nonce check, capability check, and input sanitisation?
- **Tests**: If the task mentions a test, does that test file exist and does it contain relevant assertions?

Don't run tests or execute code. This is a read-only audit.

### Step 3: Categorise each task

Assign each task one of these statuses:

- **Done** — The code exists, the implementation is complete, and it matches what the task described.
- **Partial** — Some code exists but the implementation is incomplete. State specifically what's missing (e.g. "function exists but no DocBlock", "endpoint registered but permission callback returns true unconditionally", "test file exists but no assertions").
- **Not Started** — No code exists for this task regardless of checkbox state.
- **Checkbox Mismatch** — The checkbox says done but the code says otherwise, or the checkbox says not done but the code is actually complete. Flag these clearly.

### Step 4: Produce the report

Output the report directly in the conversation. Use this format:

```
## Plugin Status Report

**Task file**: [path]
**Audited**: [date/time]
**Plugin directory**: [path]

### Summary

| Status          | Count |
|-----------------|-------|
| Done            | X     |
| Partial         | X     |
| Not Started     | X     |
| Mismatched      | X     |
| **Total**       | X     |

**Progress: X/Y tasks complete (Z%)**

### Issues Found

[List any checkbox mismatches, security gaps, missing DocBlocks, or stubs pretending to be implementations. Be specific — file path, line number if possible, and what needs fixing.]

### Remaining Work

[For each task that is Partial or Not Started, list the task number, title, and a brief note on what still needs doing. Order them in the sequence they should be tackled.]

### Done

[List completed task numbers and titles. Keep it brief — a simple numbered list.]
```

## Rules

- **Don't modify any files.** This is a read-only command. Don't update checkboxes, don't fix code, don't create files.
- **Be honest.** If a checkbox says done but the code is a stub, call it out. The whole point is an accurate picture.
- **Be specific about what's missing.** "Partial" with no explanation is useless. Say exactly what's incomplete.
- **Check security properly.** If a task involves user input, form handling, AJAX, or REST, verify nonces, capability checks, sanitisation, and escaping are actually present. A missing nonce check on a form handler is a finding, not a nitpick.
- **Check DocBlocks.** If functions are missing DocBlocks, flag it as part of the task's status. The code style rules in the main skill require them on every function.
- **Don't suggest new features or refactors.** Stick to what the task list defines. If something looks wrong architecturally, mention it briefly in a "Notes" section at the end but don't let it dominate the report.
