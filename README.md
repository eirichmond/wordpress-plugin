# WordPress Plugin Developer — Claude Code Plugin

A [Claude Code plugin](https://docs.anthropic.com/en/docs/claude-code/plugins) that gives Claude deep knowledge of WordPress plugin development. When installed, Claude writes WordPress plugins that follow modern PHP practices, WordPress Coding Standards, and established security patterns — without you having to repeat the same instructions every session.

## What It Does

This plugin adds three things to your Claude Code sessions:

### 1. WordPress Plugin Developer Skill

Automatically activates when you're working on WordPress plugin code. Claude will:

- Write PHP 8.2+ with strict types, typed properties, union types, enums, and readonly properties
- Follow WordPress Coding Standards (WPCS) — tabs, Yoda conditions, spaces inside parentheses, proper naming
- Apply security best practices — input sanitisation, output escaping, nonce verification, capability checks, prepared database queries
- Use the correct WordPress APIs — hooks, custom post types, `register_post_meta()`, block editor, REST API, WP-CLI
- Structure plugins with PSR-4 autoloading, Composer, and `@wordpress/scripts`
- Add DocBlocks to every function, favour early returns, keep functions small and single-purpose

### 2. `/plugin-tasks` Command

Generates a structured, sequential task list before any code is written.

```
/plugin-tasks A WooCommerce extension that adds gift wrapping options to products
```

This produces a `TASKS.md` file with numbered, grouped tasks where each task has a clear description and a verification step. Tasks are ordered so each one builds on the last and can be tested independently.

### 3. `/plugin-build` Command

Works through a task list one step at a time — implementing, verifying, and marking each task complete before moving on.

```
/plugin-build
```

By default it reads `TASKS.md`. You can pass a specific file if you have multiple task lists:

```
/plugin-build TASKS-gift-wrapping.md
```

After each task, Claude pauses to check in before continuing. This gives you a chance to review the code, suggest changes, or adjust direction.

## Installation

### Option 1: Clone to the plugins directory

```bash
# Create the plugins directory if it doesn't exist
mkdir -p ~/.claude/plugins

# Clone the plugin
git clone <repo-url> ~/.claude/plugins/wordpress-plugin
```

### Option 2: Clone anywhere and symlink

```bash
git clone <repo-url> ~/projects/wordpress-plugin
ln -s ~/projects/wordpress-plugin ~/.claude/plugins/wordpress-plugin
```

### Verify installation

Start a Claude Code session and the plugin should be loaded automatically. You can verify by asking Claude about WordPress plugin development patterns — it should reference WPCS, strict types, security practices, and the task-driven workflow without being prompted.

## Usage

### Starting a new plugin from scratch

1. Navigate to your plugin directory (or create an empty one):

```bash
mkdir ~/projects/my-plugin && cd ~/projects/my-plugin
```

2. Start Claude Code and describe what you want to build:

```
/plugin-tasks A WordPress plugin that adds a custom "Projects" post type with
portfolio fields, a filterable archive page, and a Gutenberg block for
displaying featured projects
```

3. Review the generated `TASKS.md`. Edit it if you want to add, remove, or reorder tasks.

4. Start building:

```
/plugin-build
```

5. Claude implements one task at a time, verifies it, marks it complete, and asks if you're ready to continue.

### Adding a feature to an existing plugin

The same workflow applies. Navigate to your existing plugin directory and use `/plugin-tasks` to plan the new feature. Claude will check your existing codebase first to avoid duplicating code or contradicting your architecture.

```
/plugin-tasks Add a REST API endpoint for the Projects post type with full CRUD,
filtering by taxonomy, and batch operations
```

### Working without the task workflow

You don't have to use the commands. The skill activates automatically whenever you're working on WordPress plugin code. Just describe what you need:

```
Add a settings page under the Settings menu with fields for API key, sync interval,
and debug mode. Use the Settings API.
```

Claude will write the code following all the same standards — strict types, WPCS, security, DocBlocks, small functions — whether you use the structured task workflow or not.

## What's Inside

### Skill

| File | Purpose |
|---|---|
| `skills/wordpress-plugin-developer/SKILL.md` | Core skill definition — coding standards, security rules, plugin structure, patterns for hooks, CPTs, blocks, REST API, assets, i18n, testing, and performance |

### Commands

| Command | File | Purpose |
|---|---|---|
| `/plugin-tasks` | `commands/plugin-tasks/SKILL.md` | Generate a sequential task list from a feature description |
| `/plugin-build` | `commands/plugin-build/SKILL.md` | Execute a task list step-by-step with verification |
| `/plugin-status` | `commands/plugin-status/SKILL.md` | *(Planned)* |

### Reference Files

The skill uses progressive disclosure — core patterns are in the main skill file, and detailed templates and examples are in reference files that Claude loads only when needed:

| File | Contents |
|---|---|
| `references/bootstrap.md` | Full main plugin file template, `composer.json`, `phpcs.xml.dist`, `package.json` |
| `references/security.md` | Input sanitisation, output escaping, nonce verification, capability checks, `$wpdb->prepare()` |
| `references/blocks.md` | `block.json` template, `edit.js`/`save.js` patterns, `render.php` for dynamic blocks, Interactivity API |
| `references/rest-api.md` | Full `WP_REST_Controller` example with CRUD routes, permission callbacks, schema |
| `references/testing.md` | PHPUnit config, `.wp-env.json`, `WP_UnitTestCase` examples, Playwright e2e setup |
| `references/lifecycle.md` | Activator/Deactivator classes, `dbDelta()`, cron scheduling, `uninstall.php`, WP-CLI commands |

## Target Environment

Plugins built with this tool target:

- **PHP 8.2+** — strict types, typed properties, enums, readonly, match expressions, named arguments
- **WordPress 6.7+** — current APIs only, no deprecated functions
- **Composer** — PSR-4 autoloading, PHPCS, PHPUnit, PHPStan as dev dependencies
- **Node/npm** — `@wordpress/scripts` for block compilation and asset bundling
- **wp-env** — local development and test environment

## Plugin Structure

Plugins are scaffolded with this structure:

```
plugin-name/
├── plugin-name.php          # Thin bootstrap: header, constants, autoloader, boot
├── composer.json             # PSR-4 autoloading, dev deps (PHPCS, PHPUnit, PHPStan)
├── package.json              # @wordpress/scripts
├── uninstall.php             # Clean data removal
├── .wp-env.json              # Local dev environment
├── phpunit.xml.dist          # Test config
├── phpcs.xml.dist            # WPCS ruleset
├── src/                      # PHP source (PSR-4 root)
│   ├── Plugin.php            # Main plugin class (singleton)
│   ├── Admin/                # Admin screens, settings pages
│   ├── Frontend/             # Public-facing output
│   ├── PostTypes/            # Custom post type registrations
│   ├── Taxonomies/           # Custom taxonomies
│   ├── Blocks/               # Server-side block logic
│   ├── REST/                 # REST API controllers
│   ├── CLI/                  # WP-CLI commands
│   └── Services/             # Business logic
├── src-blocks/               # Block source (JS/CSS per block)
│   └── example-block/
│       ├── block.json
│       ├── edit.js, save.js, index.js
│       ├── editor.scss, style.scss
│       └── view.js           # Interactivity API store
├── build/                    # Compiled assets (gitignored)
├── assets/                   # Static CSS/JS/images
├── languages/                # Translation files
├── templates/                # PHP template partials
├── tests/
│   ├── php/
│   │   ├── Unit/             # Pure PHP tests, no WordPress
│   │   ├── Integration/      # Tests using WP test framework
│   │   └── bootstrap.php
│   └── e2e/                  # Playwright tests
└── vendor/                   # Composer deps (gitignored)
```

## Contributing

To modify the plugin's behaviour:

- **Coding standards and patterns**: Edit `skills/wordpress-plugin-developer/SKILL.md`
- **Detailed examples and templates**: Edit the relevant file in `references/`
- **Task generation workflow**: Edit `commands/plugin-tasks/SKILL.md`
- **Build execution workflow**: Edit `commands/plugin-build/SKILL.md`
- **Plugin metadata**: Edit `.claude-plugin/plugin.json`

Skill and command files use YAML frontmatter for metadata (`name`, `description`, `allowed-tools`, etc.) followed by markdown instructions that Claude follows at runtime.

## License

GPL-2.0-or-later
