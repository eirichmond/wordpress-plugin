---
name: plugin-tasks
description: Generate a structured task list for building a WordPress plugin feature or entire plugin. Reads a PRD file produced by /plugin-plan and outputs a markdown file with sequential, testable steps.
disable-model-invocation: true
argument-hint: <path-to-prd-file>
allowed-tools: Read, Write, Bash, Grep, Glob
---

# Generate WordPress Plugin Task List

You are generating a task list for a WordPress plugin development project. You will read a PRD (Product Requirements Document) and translate it into a structured, sequential set of build tasks.

## Input

PRD file to use: **$ARGUMENTS**

If no argument is provided, look for `PRD.md` in the project root. If multiple `PRD-*.md` files exist, list them and ask the user which one to use.

If the argument is not a file path (i.e. it looks like a plain description rather than a filename), that's fine — treat it as a direct description and proceed. But tell the user that for better results, they should run `/plugin-plan` first to produce a PRD.

## Process

1. **Read the PRD.** Load the full PRD into context. Understand the plugin's purpose, data model, features, integrations, lifecycle, and testing strategy.

2. **Research the codebase.** Before writing tasks, check the current project directory for existing code, an existing CLAUDE.md, composer.json, package.json, or any prior task files. Understand what already exists so you don't duplicate work or contradict existing architecture.

3. **Write the task list.** Produce a markdown file at `TASKS.md` in the project root (or if one already exists, create `TASKS-<slugified-feature-name>.md`).

## Task List Format

Use this exact structure:

```markdown
# Task List: [Feature/Plugin Name]

> Generated from: [PRD file path or direct description]
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
- Any PRD sections that were ambiguous or produced assumptions — flag these so the developer can verify.
- Suggest they review it and adjust before starting, then use `/plugin-build` to begin working through it.