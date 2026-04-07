---
name: plugin-tasks
description: Generate a structured task list for building a WordPress plugin feature or entire plugin. Outputs a markdown file with sequential, testable steps.
disable-model-invocation: true
argument-hint: <description of the plugin or feature to build>
allowed-tools: Read, Write, Bash
---

# Generate WordPress Plugin Task List

You are generating a task list for a WordPress plugin development project. The user will describe what they want to build and you will produce a structured, sequential task list saved as a markdown file.

## Input

The user's description of what to build: **$ARGUMENTS**

## Process

1. **Analyse the request.** Break down what the user described into logical areas of functionality. Think about what data layer is needed, what admin UI is required, what public-facing output exists, what APIs are involved, and what tests should cover.

2. **Research the codebase.** Before writing tasks, check the current project directory for existing code, an existing CLAUDE.md, composer.json, package.json, or any prior task files. Understand what already exists so you don't duplicate work or contradict existing architecture.

3. **Write the task list.** Produce a markdown file at `TASKS.md` in the project root (or if one already exists, create `TASKS-<slugified-feature-name>.md`).

## Task List Format

Use this exact structure:

```markdown
# Task List: [Feature/Plugin Name]

> Generated from: [the user's original description]
> Date: [today's date]

## Overview

[2-3 sentences explaining what will be built and the high-level approach.]

## Prerequisites

- [ ] [Any setup steps needed before development starts, e.g. "Run composer install", "Ensure wp-env is configured"]

## 1. [First Logical Group — e.g. "Data Layer"]

- [ ] **1.1** [Short task title]
  - What: [One sentence explaining the work]
  - Test: [How to verify this step works before moving on]

- [ ] **1.2** [Short task title]
  - What: [One sentence explaining the work]
  - Test: [How to verify this step works before moving on]

## 2. [Second Logical Group — e.g. "Admin UI"]

- [ ] **2.1** [Short task title]
  - What: [One sentence explaining the work]
  - Test: [How to verify this step works before moving on]

## [Continue numbering groups...]

## Final Checks

- [ ] Run PHPCS — `composer lint`
- [ ] Run PHPStan — `composer analyse`
- [ ] Run PHPUnit — run via wp-env
- [ ] Run Playwright e2e tests if applicable
- [ ] Manual smoke test in browser
```

## Rules

- **Every task must be testable in isolation.** If a task can't be verified on its own, break it down further.
- **Tasks must be sequential.** No task should depend on something that comes later in the list.
- **Keep tasks small.** Each task should be completable in a single focused session. If a task description needs more than two sentences, it's too big — split it.
- **Group logically.** Common groups: Data Layer, Admin UI, Frontend Output, REST API, Block Editor, CLI Commands, Testing, Documentation. Only include groups relevant to the feature.
- **Include the test for each task.** This can be "verify in admin", "run PHPUnit test X", "check REST endpoint returns 200", "confirm block renders in editor", etc. The point is that the developer knows exactly how to confirm the step is done before moving on.
- **Include file paths where possible.** If a task creates or modifies a file, mention which file.
- **Don't include boilerplate setup tasks that already exist.** Check the project first.

## After Generating

Once the file is saved, tell the user:
- The file path where the task list was saved.
- The total number of tasks.
- Suggest they review it and adjust before starting, then use `/plugin-build` to begin working through it.
