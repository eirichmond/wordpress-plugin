# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Is

This is a **Claude Code plugin** (`wordpress-plugin`) that provides skills, commands, and reference material for WordPress plugin development. It is NOT a WordPress plugin itself — it's tooling that teaches Claude Code how to build WordPress plugins properly.

## Architecture

```
.claude-plugin/plugin.json   # Plugin manifest (name, version, author)
skills/
  wordpress-plugin-developer/SKILL.md  # Core skill: WP plugin dev standards, patterns, code style
commands/
  plugin-tasks/SKILL.md      # /plugin-tasks — generates sequential task lists (TASKS.md)
  plugin-build/SKILL.md      # /plugin-build — works through task lists step-by-step
  plugin-status/SKILL.md     # /plugin-status (currently empty)
references/                  # Progressive disclosure: detailed examples loaded on demand
  bootstrap.md               # Main plugin file, composer.json, phpcs.xml.dist, package.json
  security.md                # Nonces, capabilities, sanitisation, escaping, $wpdb->prepare()
  blocks.md                  # block.json, edit.js/save.js, render.php, Interactivity API
  rest-api.md                # Full WP_REST_Controller CRUD example
  testing.md                 # PHPUnit, wp-env, WP_UnitTestCase, Playwright
  lifecycle.md               # Activation, deactivation, uninstall, dbDelta, cron, WP-CLI
```

## Key Design Decisions

- **Progressive disclosure**: The main skill SKILL.md covers core patterns; reference files are only loaded when a specific task needs that detail. Don't load all references at once.
- **Task-driven workflow**: The plugin enforces a two-step workflow — first `/plugin-tasks` to plan, then `/plugin-build` to execute one task at a time with verification between each step.
- **Target environment**: PHP 8.2+, WordPress 6.7+, `@wordpress/scripts` for blocks, Composer for autoloading/dev deps.
- **Code style**: WordPress Coding Standards (WPCS) — tabs, Yoda conditions, spaces inside parentheses, `declare(strict_types=1)` on every PHP file, DocBlocks on every function.

## Working on This Plugin

When modifying skill or command definitions:
- SKILL.md files use YAML frontmatter (`name`, `description`, `disable-model-invocation`, `argument-hint`, `allowed-tools`)
- Commands live in `commands/<name>/SKILL.md` and are invoked as `/<name>`
- Reference files are plain markdown with code examples — they're read by the skill at runtime, not by the plugin system
- The `plugin.json` manifest defines the plugin identity
