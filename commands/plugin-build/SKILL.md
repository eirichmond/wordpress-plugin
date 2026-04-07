---
name: plugin-build
description: Work through a WordPress plugin task list step by step, building and testing each task before moving to the next. Reads from TASKS.md or a specified task file.
disable-model-invocation: true
argument-hint: [path-to-tasks-file]
allowed-tools: Read, Write, Edit, Bash, Grep, Glob
---

# Build WordPress Plugin From Task List

You are building a WordPress plugin by working through a task list one step at a time. You will implement each task, verify it works, then move to the next.

## Input

Task file to use: **$ARGUMENTS**

If no argument is provided, look for `TASKS.md` in the project root. If multiple `TASKS-*.md` files exist, list them and ask the user which one to work from.

## Before Starting

1. **Read the task file.** Load the full task list into context.
2. **Read the SKILL.md** in this plugin's skill directory to load the WordPress plugin development standards. Follow all coding standards, security rules, and code style rules defined there.
3. **Check the codebase.** Look at what already exists — files, composer.json, package.json, existing src/ structure. Understand the current state.
4. **Find the first unchecked task.** Scan for the first `- [ ]` item. That's where you start.

## Workflow — Repeat For Each Task

For every task, follow this exact sequence:

### Step 1: Announce the task

Tell the user which task you're starting. State the task number, title, and what you're about to do. Keep it brief.

### Step 2: Implement the task

Write the code. Follow these rules from the WordPress Plugin Developer skill:

- **Small functions.** Each function does one thing. If it's doing two things, split it.
- **DocBlocks on every function.** `@param`, `@return`, `@throws` — no exceptions.
- **Early returns over nesting.** Guard clauses first, happy path after.
- **Security.** Sanitise input, escape output, verify nonces, check capabilities. Every time.
- **WordPress coding standards.** Tabs, Yoda conditions, spaces inside parentheses, proper naming conventions.
- **Strict types.** `declare(strict_types=1);` at the top of every PHP file.

### Step 3: Verify the task

Run the test described in the task list. This might mean:
- Running a specific PHPUnit test
- Running a WP-CLI command to check data
- Checking that a file was created correctly
- Running PHPCS on the new code
- Confirming an endpoint returns the expected response

If the task list says "verify X appears in admin", you obviously can't do that — tell the user to check it manually and confirm before you continue.

If verification fails, fix the issue before moving on. Do not skip to the next task with a broken step behind you.

### Step 4: Mark the task complete

Update the task file. Change `- [ ]` to `- [x]` for the completed task. Save the file.

### Step 5: Pause and check in

After completing each task, briefly tell the user:
- What you did.
- Whether verification passed.
- What the next task is.

Then ask: **"Ready for the next task, or do you want to review first?"**

Wait for the user to confirm before proceeding. Do not chain multiple tasks without checking in.

## Rules

- **One task at a time.** Never implement two tasks in the same step. The whole point is sequential, testable progress.
- **Don't skip tasks.** If a task seems unnecessary, flag it to the user rather than skipping it.
- **Don't reorder tasks.** The task list was designed to be sequential. If you think the order is wrong, tell the user before changing anything.
- **Keep the task file updated.** The checked/unchecked state in the markdown file should always reflect reality.
- **If you hit a blocker,** stop and explain the issue clearly. Don't work around it silently. Common blockers: missing dependency, ambiguous requirement, conflicting existing code.
- **Reference the skill's reference files when needed.** If you're building a REST controller, read `references/rest-api.md`. If you're setting up tests, read `references/testing.md`. Load what you need, when you need it.

## When All Tasks Are Complete

Once every task is checked off:

1. Run the full verification suite: PHPCS, PHPStan, PHPUnit.
2. Summarise what was built — list the files created/modified.
3. Note any manual testing the user should do.
4. Suggest any follow-up tasks that became apparent during the build (but don't add them to the task file without asking).
