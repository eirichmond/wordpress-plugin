# Bootstrap Reference

Full templates for the main plugin file, Composer configuration, PHPCS ruleset, and package.json.

## Main Plugin File (Complete)

```php
<?php
/**
 * Plugin Name:       Plugin Name
 * Plugin URI:        https://example.com/plugin-name
 * Description:       Brief description of the plugin.
 * Version:           1.0.0
 * Requires at least: 6.7
 * Requires PHP:      8.2
 * Author:            Author Name
 * Author URI:        https://example.com
 * License:           GPL-2.0-or-later
 * License URI:       https://www.gnu.org/licenses/gpl-2.0.html
 * Text Domain:       plugin-name
 * Domain Path:       /languages
 * Update URI:        https://example.com/plugin-name
 */

declare(strict_types=1);

namespace PluginName;

// Prevent direct access.
if ( ! defined( 'ABSPATH' ) ) {
    exit;
}

// Plugin constants.
define( 'PLUGIN_NAME_VERSION', '1.0.0' );
define( 'PLUGIN_NAME_FILE', __FILE__ );
define( 'PLUGIN_NAME_DIR', plugin_dir_path( __FILE__ ) );
define( 'PLUGIN_NAME_URL', plugin_dir_url( __FILE__ ) );
define( 'PLUGIN_NAME_BASENAME', plugin_basename( __FILE__ ) );

// Composer autoloader.
if ( file_exists( __DIR__ . '/vendor/autoload.php' ) ) {
    require_once __DIR__ . '/vendor/autoload.php';
} else {
    // If Composer autoload is missing, the plugin can't function.
    add_action( 'admin_notices', static function (): void {
        echo '<div class="notice notice-error"><p>';
        echo esc_html__( 'Plugin Name requires Composer dependencies. Run `composer install`.', 'plugin-name' );
        echo '</p></div>';
    } );
    return;
}

// Boot the plugin after all plugins are loaded.
add_action( 'plugins_loaded', static function (): void {
    Plugin::get_instance();
} );

// Activation and deactivation hooks MUST be in the main file.
register_activation_hook( __FILE__, [ Activator::class, 'activate' ] );
register_deactivation_hook( __FILE__, [ Deactivator::class, 'deactivate' ] );
```

## Composer Configuration

```json
{
    "name": "vendor/plugin-name",
    "description": "Plugin description.",
    "type": "wordpress-plugin",
    "license": "GPL-2.0-or-later",
    "require": {
        "php": ">=8.2"
    },
    "require-dev": {
        "wp-coding-standards/wpcs": "^3.0",
        "phpcompatibility/phpcompatibility-wp": "*",
        "dealerdirect/phpcodesniffer-composer-installer": "^1.0",
        "phpunit/phpunit": "^10.0",
        "phpstan/phpstan": "^1.0",
        "szepeviktor/phpstan-wordpress": "^1.0",
        "yoast/phpunit-polyfills": "^2.0"
    },
    "autoload": {
        "psr-4": {
            "PluginName\\": "src/"
        }
    },
    "autoload-dev": {
        "psr-4": {
            "PluginName\\Tests\\": "tests/php/"
        }
    },
    "config": {
        "allow-plugins": {
            "dealerdirect/phpcodesniffer-composer-installer": true
        }
    },
    "scripts": {
        "lint": "phpcs",
        "lint:fix": "phpcbf",
        "test": "phpunit",
        "analyse": "phpstan analyse"
    }
}
```

## PHPCS Ruleset (phpcs.xml.dist)

```xml
<?xml version="1.0"?>
<ruleset name="Plugin Name">
    <description>PHPCS ruleset for Plugin Name.</description>

    <file>./src</file>
    <file>./plugin-name.php</file>
    <file>./uninstall.php</file>

    <arg name="extensions" value="php"/>
    <arg name="colors"/>
    <arg value="sp"/>

    <!-- Use WordPress coding standards -->
    <rule ref="WordPress">
        <!-- Exclude filename rules when using PSR-4 autoloading -->
        <exclude name="WordPress.Files.FileName.InvalidClassFileName"/>
        <exclude name="WordPress.Files.FileName.NotHyphenatedLowercase"/>
    </rule>

    <!-- Set the text domain for i18n checks -->
    <rule ref="WordPress.WP.I18n">
        <properties>
            <property name="text_domain" type="array">
                <element value="plugin-name"/>
            </property>
        </properties>
    </rule>

    <!-- Minimum WordPress version for deprecated function checks -->
    <config name="minimum_wp_version" value="6.7"/>

    <!-- PHP compatibility checks -->
    <rule ref="PHPCompatibilityWP"/>
    <config name="testVersion" value="8.2-"/>
</ruleset>
```

## PHPStan Configuration (phpstan.neon)

```neon
includes:
    - vendor/szepeviktor/phpstan-wordpress/extension.neon

parameters:
    level: 8
    paths:
        - src/
        - plugin-name.php
    scanDirectories:
        - vendor/
    ignoreErrors: []
```

## Package.json (for block development)

```json
{
    "name": "plugin-name",
    "version": "1.0.0",
    "private": true,
    "scripts": {
        "build": "wp-scripts build --webpack-src-dir=src-blocks --output-path=build",
        "start": "wp-scripts start --webpack-src-dir=src-blocks --output-path=build",
        "lint:js": "wp-scripts lint-js src-blocks/",
        "lint:css": "wp-scripts lint-style 'src-blocks/**/*.scss'",
        "test:e2e": "wp-scripts test-playwright",
        "plugin-zip": "wp-scripts plugin-zip"
    },
    "devDependencies": {
        "@wordpress/scripts": "^30.0.0",
        "@wordpress/e2e-test-utils-playwright": "^1.0.0"
    }
}
```

## Plugin Class Pattern

```php
<?php

declare(strict_types=1);

namespace PluginName;

final class Plugin {

    private static ?self $instance = null;

    public static function get_instance(): self {
        if ( null === self::$instance ) {
            self::$instance = new self();
        }
        return self::$instance;
    }

    private function __construct() {
        $this->register_hooks();
    }

    private function register_hooks(): void {
        add_action( 'init', [ $this, 'register_post_types' ] );
        add_action( 'init', [ $this, 'register_blocks' ] );
        add_action( 'rest_api_init', [ $this, 'register_rest_routes' ] );
        add_action( 'admin_menu', [ $this, 'register_admin_pages' ] );
        add_action( 'wp_enqueue_scripts', [ $this, 'enqueue_frontend_assets' ] );
        add_action( 'admin_enqueue_scripts', [ $this, 'enqueue_admin_assets' ] );
        add_action( 'init', [ $this, 'load_textdomain' ] );
    }

    public function load_textdomain(): void {
        load_plugin_textdomain(
            'plugin-name',
            false,
            dirname( PLUGIN_NAME_BASENAME ) . '/languages'
        );
    }

    public function register_post_types(): void {
        // Delegate to PostTypes classes.
    }

    public function register_blocks(): void {
        // register_block_type() calls.
    }

    public function register_rest_routes(): void {
        // Instantiate and register REST controllers.
    }

    public function register_admin_pages(): void {
        // add_menu_page() / add_submenu_page() calls.
    }

    public function enqueue_frontend_assets(): void {
        // Conditional wp_enqueue_style/script.
    }

    public function enqueue_admin_assets( string $hook_suffix ): void {
        // Only load on plugin's own admin pages.
    }
}
```
