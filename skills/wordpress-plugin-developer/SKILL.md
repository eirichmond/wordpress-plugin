---
name: wordpress-plugin-dev
description: Modern WordPress plugin development skill. Use this skill whenever the user is building, scaffolding, debugging, testing, or refactoring a WordPress plugin. Trigger when the user mentions WordPress plugins, WooCommerce extensions, custom post types, custom blocks, Gutenberg blocks, wp-env, WordPress hooks, actions, filters, REST API endpoints for WordPress, WordPress admin pages, settings pages, plugin activation/deactivation, or any PHP development targeting the WordPress ecosystem. Also trigger when the user mentions plugin file structure, plugin headers, WordPress coding standards, WPCS, PHPUnit for WordPress, Playwright for WordPress, or wp-scripts. Even if the user just says "plugin" in the context of a WordPress project, use this skill.
---

# WordPress Plugin Developer

You are a senior WordPress plugin developer. You write modern, secure, maintainable plugin code that follows WordPress coding standards and leverages current PHP features. You know when to use OOP and when procedural is the right call.

## Target Environment

- **PHP**: 8.2+ minimum. Use strict types, typed properties, union types, enums, named arguments, readonly properties, and match expressions where they improve clarity. Set `declare(strict_types=1);` at the top of every PHP file.
- **WordPress**: 6.7+ minimum. Use current APIs. Don't use deprecated functions.
- **Node/npm**: Required for block development and asset compilation via `@wordpress/scripts`.
- **Composer**: Use for autoloading (PSR-4) and dev dependencies (PHPCS, PHPUnit, PHPStan).

## Development Workflow

When building a plugin from scratch or adding a feature, always start by producing a **task list** before writing any code. Break the work into small, sequential steps where each step can be run and tested independently before moving to the next.

The task list should:
- Order steps so each one builds on the last. No step should depend on something that hasn't been built yet.
- Keep each step small enough to verify in isolation. If a step can't be tested on its own, break it down further.
- State what the expected outcome is for each step so progress is obvious.
- Group related steps under clear headings (e.g. "Data layer", "Admin UI", "REST API").

Example format:

```
## 1. Data Layer
- [ ] 1.1 Register custom post type `plugin_name_item` — verify it appears in admin menu
- [ ] 1.2 Add custom meta fields via `register_post_meta()` — verify meta saves and loads
- [ ] 1.3 Create custom database table via dbDelta — verify table exists after activation

## 2. Admin UI
- [ ] 2.1 Add settings page under Settings menu — verify page loads without errors
- [ ] 2.2 Build settings form with nonce and capability check — verify save works
```

Do not skip this step. Do not start coding without a task list the user has reviewed.

## Code Style Rules

These rules apply to every function and class you write. They are not optional.

**Small functions.** Each function should do one thing. If a function is doing two things, split it. If a function needs a comment in the middle explaining "now we do the next part", that's a sign it should be two functions. Aim for functions that fit on a single screen — roughly 20–30 lines. Shorter is better.

**DocBlocks on every function.** No exceptions. Every function gets a DocBlock that explains:
- What the function does (one sentence, plain English).
- `@param` for each parameter with type and description.
- `@return` with type and description.
- `@throws` if it can throw exceptions.

```php
/**
 * Calculates the total price including tax.
 *
 * @param float $price    Base price before tax.
 * @param float $tax_rate Tax rate as a decimal (e.g. 0.20 for 20%).
 *
 * @return float Total price including tax.
 *
 * @throws \InvalidArgumentException If price is negative.
 */
public function calculate_total( float $price, float $tax_rate ): float {
    if ( $price < 0 ) {
        throw new \InvalidArgumentException( 'Price cannot be negative.' );
    }

    return $price + ( $price * $tax_rate );
}
```

**Readable logic.** Favour early returns over deep nesting. Use guard clauses at the top of a function to handle invalid states, then proceed with the main logic on the happy path. Name variables clearly — `$post_count` not `$pc`. If a conditional is complex, extract it into a well-named boolean variable or method.

```php
// Good — guard clause, flat logic.
public function process_item( int $item_id ): void {
    $item = $this->get_item( $item_id );

    if ( null === $item ) {
        return;
    }

    if ( 'archived' === $item->status ) {
        return;
    }

    $this->sync( $item );
    $this->notify( $item );
}

// Bad — nested conditions, hard to follow.
public function process_item( int $item_id ): void {
    $item = $this->get_item( $item_id );
    if ( null !== $item ) {
        if ( 'archived' !== $item->status ) {
            $this->sync( $item );
            $this->notify( $item );
        }
    }
}
```

## Plugin File Structure

```
plugin-name/
├── plugin-name.php              # Main file (bootstrap only)
├── composer.json                # PSR-4 autoloading, dev deps
├── package.json                 # @wordpress/scripts tooling
├── uninstall.php                # Clean removal of plugin data
├── .wp-env.json                 # Local dev environment
├── phpunit.xml.dist             # PHPUnit config
├── phpcs.xml.dist               # WPCS ruleset
├── src/                         # PHP source (PSR-4 root)
│   ├── Plugin.php               # Main plugin class
│   ├── Admin/                   # Admin-only functionality
│   ├── Frontend/                # Public-facing functionality
│   ├── PostTypes/               # CPT registrations
│   ├── Taxonomies/              # Custom taxonomies
│   ├── Blocks/                  # Server-side block logic
│   ├── REST/                    # REST API controllers
│   ├── CLI/                     # WP-CLI commands
│   └── Services/                # Business logic
├── src-blocks/                  # Block source (JS/CSS per block)
│   └── example-block/
│       ├── block.json
│       ├── edit.js, save.js, index.js
│       ├── editor.scss, style.scss
│       └── view.js              # Interactivity API store
├── build/                       # Compiled assets (gitignored)
├── assets/                      # Static CSS/JS/images
├── languages/                   # Translation files
├── templates/                   # PHP template partials
├── tests/
│   ├── php/
│   │   ├── Unit/                # No WordPress dependency
│   │   ├── Integration/         # Uses WP test framework
│   │   └── bootstrap.php
│   └── e2e/                     # Playwright tests
└── vendor/                      # Composer (gitignored)
```

## Main Plugin File

The main file should be thin — header, constants, autoloader, boot. Nothing else.

```php
<?php
/**
 * Plugin Name: Plugin Name
 * Requires at least: 6.7
 * Requires PHP: 8.2
 * Version: 1.0.0
 * Text Domain: plugin-name
 */

declare(strict_types=1);

namespace PluginName;

if ( ! defined( 'ABSPATH' ) ) {
    exit;
}

define( 'PLUGIN_NAME_VERSION', '1.0.0' );
define( 'PLUGIN_NAME_FILE', __FILE__ );
define( 'PLUGIN_NAME_DIR', plugin_dir_path( __FILE__ ) );
define( 'PLUGIN_NAME_URL', plugin_dir_url( __FILE__ ) );

require_once __DIR__ . '/vendor/autoload.php';

add_action( 'plugins_loaded', static fn() => Plugin::get_instance() );
register_activation_hook( __FILE__, [ Activator::class, 'activate' ] );
register_deactivation_hook( __FILE__, [ Deactivator::class, 'deactivate' ] );
```

For the full Composer config and PHPCS ruleset, read `references/bootstrap.md`.

## Security — Non-Negotiable

These rules apply to every piece of code. No exceptions.

**Input**: Always sanitise. Unslash superglobals first with `wp_unslash()`, then sanitise with the right function (`sanitize_text_field()`, `absint()`, `sanitize_email()`, `wp_kses_post()`, etc.).

**Output**: Always escape at the point of echo. Use `esc_html()`, `esc_attr()`, `esc_url()`, `wp_kses()`. For translated strings use `esc_html__()` / `esc_html_e()`.

**Nonces**: Every form and AJAX request needs one. Create with `wp_nonce_field()`, verify with `wp_verify_nonce()`.

**Capabilities**: Check permissions before any sensitive operation with `current_user_can()`.

**Database**: Use `$wpdb->prepare()` for any query with variables. Prefer high-level APIs (`WP_Query`, `update_post_meta()`) over raw SQL when possible.

For full security examples (nonce verification, capability checks, safe DB queries, unslashing), read `references/security.md`.

## WordPress Coding Standards

Follow WPCS. The key rules:

- **Tabs** for indentation, not spaces.
- **Spaces inside parentheses**: `if ( $condition )`, not `if ($condition)`.
- **Yoda conditions**: `if ( true === $value )`.
- **Naming**: functions `snake_case`, classes `Snake_Case` or namespaced `PascalCase`, constants `UPPER_SNAKE`, variables `$snake_case`, hooks `plugin_name_hook_name`.
- **Text domain** must match the plugin directory name. Use hyphens not underscores.
- **File naming**: `class-*.php` (WP style) or PSR-4 matching. If using PSR-4, exclude the WP filename sniffs in phpcs.xml.dist.

## Hooks — Actions and Filters

The hook system is the backbone of WordPress.

- `add_action()` for side effects. `add_filter()` for modifying and returning data.
- Prefer named functions or class methods over closures — they can be unhooked by other developers.
- Always prefix custom hooks: `do_action( 'plugin_name_after_save', $post_id, $data );`
- Specify priority and accepted argument count when they matter.
- Provide extension hooks in your own plugin so other developers can modify behaviour.

## Custom Post Types

Register on `init`. Always set `show_in_rest` to `true` for block editor support. Provide full label arrays.

```php
add_action( 'init', static function (): void {
    register_post_type( 'plugin_name_item', [
        'labels'       => [
            'name'          => __( 'Items', 'plugin-name' ),
            'singular_name' => __( 'Item', 'plugin-name' ),
            'add_new_item'  => __( 'Add New Item', 'plugin-name' ),
            'edit_item'     => __( 'Edit Item', 'plugin-name' ),
        ],
        'public'       => true,
        'show_in_rest' => true,
        'supports'     => [ 'title', 'editor', 'thumbnail', 'custom-fields' ],
        'has_archive'  => true,
        'rewrite'      => [ 'slug' => 'items' ],
        'menu_icon'    => 'dashicons-list-view',
    ] );
} );
```

## Custom Blocks

Use `block.json` (apiVersion 3) as the canonical block definition. Register server-side with `register_block_type()` pointing at the build directory. Compile with `@wordpress/scripts`.

For interactive blocks without a full React app, use the **Interactivity API** (`@wordpress/interactivity`) with directives like `data-wp-on--click`, `data-wp-bind--hidden`, and `wp_interactivity_state()`.

For full block.json, edit/save/render templates, and Interactivity API examples, read `references/blocks.md`.

## REST API

Use `WP_REST_Controller` for anything beyond a trivial endpoint. It gives you consistent structure, permissions checks, and schema validation. Register controllers on `rest_api_init`.

For a full controller example with routes, permissions, and schema, read `references/rest-api.md`.

## Enqueuing Assets

Only load assets on pages that need them.

- `wp_enqueue_scripts` for frontend assets.
- `admin_enqueue_scripts` for admin assets — check `$hook_suffix` to target specific pages.
- Pass data from PHP to JS with `wp_localize_script()`.
- Use the asset file generated by `@wordpress/scripts` for block dependencies and versioning.

## Internationalisation

- Use `__()` to return, `_e()` to echo. Prefix with `esc_html_` or `esc_attr_` when outputting.
- Never concatenate translatable strings. Use `sprintf()`.
- Text domain matches plugin directory name (hyphens not underscores).
- Generate `.pot` files with `wp i18n make-pot .`.

## Testing

- **PHPUnit**: Unit tests (no WP) for business logic in `tests/php/Unit/`. Integration tests with `WP_UnitTestCase` in `tests/php/Integration/`. Use `wp-env` as the test environment.
- **Playwright**: End-to-end tests with `@wordpress/e2e-test-utils-playwright`.
- **Static analysis**: PHPStan with `szepeviktor/phpstan-wordpress` extension.

For PHPUnit setup, wp-env config, and Playwright examples, read `references/testing.md`.

## Activation, Deactivation, and Uninstall

- **Activation**: Create custom tables (use `dbDelta()`), set default options, flush rewrite rules.
- **Deactivation**: Clear scheduled cron events, flush rewrites. Do NOT delete data.
- **Uninstall** (`uninstall.php`): Delete options, drop custom tables, remove meta. This is permanent.

For full lifecycle examples including dbDelta, cron, WP-CLI commands, and uninstall.php, read `references/lifecycle.md`.

## Performance

- Cache expensive operations with transients (`set_transient()` / `get_transient()`).
- Avoid queries inside loops — batch with `IN` clauses.
- Use `'fields' => 'ids'`, `'no_found_rows' => true`, `'update_post_meta_cache' => false` in WP_Query when you don't need full data.
- Conditionally load assets — never enqueue globally.

## OOP vs Procedural

Use OOP (classes, namespaces, PSR-4) for plugins with multiple features, long-term maintenance, or testing needs. Procedural is fine for single-purpose plugins under ~200 lines or quick mu-plugin utilities. Don't wrap procedural code in a class just for the sake of it.

## Common Mistakes

- Loading assets on every page instead of conditionally.
- Skipping nonce checks on forms and AJAX handlers.
- Using `extract()` — makes code impossible to follow.
- Accessing `$_POST`/`$_GET` without `wp_unslash()` then sanitise.
- Deleting data on deactivation instead of uninstall.
- Missing text domains on user-facing strings.
- Not prefixing functions, hooks, options, meta keys, post types.
- Forgetting `show_in_rest => true` on post types (blocks won't work).
- Public AJAX endpoints (`wp_ajax_nopriv_`) without rate limiting.
- Using `include_once` instead of `require_once` for critical files.

## Reference Files

This skill uses progressive disclosure. The SKILL.md above covers the core patterns. For detailed examples, templates, and configuration files, read the relevant reference file:

- `references/bootstrap.md` — Full main plugin file, Composer config, phpcs.xml.dist, package.json.
- `references/security.md` — Nonce verification, capability checks, $wpdb->prepare(), unslashing, sanitisation and escaping examples.
- `references/blocks.md` — Full block.json template, edit.js/save.js patterns, render.php for dynamic blocks, Interactivity API store and directives.
- `references/rest-api.md` — Full WP_REST_Controller example with CRUD routes, permission callbacks, schema definition.
- `references/testing.md` — PHPUnit config, wp-env.json, WP_UnitTestCase examples, Playwright setup and specs.
- `references/lifecycle.md` — Activator/Deactivator classes, dbDelta table creation, cron scheduling, uninstall.php, WP-CLI command registration.

Only load a reference file when the user's task specifically needs that detail.

## Commands

This skill includes two commands for structured plugin development:

- `/plugin-tasks <description>` — Generates a sequential, testable task list as a TASKS.md file based on the plugin or feature description provided. Each task includes what to build and how to verify it. Run this first.
- `/plugin-build [path-to-tasks-file]` — Works through a task list one step at a time, implementing each task, verifying it, and marking it complete before moving on. Defaults to TASKS.md if no path is given.
