# Lifecycle Reference

Activation, deactivation, uninstall, database migrations, cron scheduling, and WP-CLI commands.

## Activator

Runs once when the plugin is activated.

```php
<?php

declare(strict_types=1);

namespace PluginName;

class Activator {

    public static function activate(): void {
        self::check_requirements();
        self::create_tables();
        self::set_default_options();

        // Flush rewrite rules after registering CPTs.
        // CPTs must be registered before this runs.
        Plugin::get_instance();
        flush_rewrite_rules();
    }

    private static function check_requirements(): void {
        if ( version_compare( PHP_VERSION, '8.2', '<' ) ) {
            deactivate_plugins( plugin_basename( PLUGIN_NAME_FILE ) );
            wp_die(
                esc_html__( 'This plugin requires PHP 8.2 or higher.', 'plugin-name' ),
                esc_html__( 'Plugin Activation Error', 'plugin-name' ),
                [ 'back_link' => true ]
            );
        }

        global $wp_version;
        if ( version_compare( $wp_version, '6.7', '<' ) ) {
            deactivate_plugins( plugin_basename( PLUGIN_NAME_FILE ) );
            wp_die(
                esc_html__( 'This plugin requires WordPress 6.7 or higher.', 'plugin-name' ),
                esc_html__( 'Plugin Activation Error', 'plugin-name' ),
                [ 'back_link' => true ]
            );
        }
    }

    private static function create_tables(): void {
        global $wpdb;

        $table_name      = $wpdb->prefix . 'plugin_name_items';
        $charset_collate = $wpdb->get_charset_collate();

        // dbDelta is picky about formatting:
        // - Each field on its own line.
        // - TWO spaces before PRIMARY KEY (not one, not a tab).
        // - KEY (not INDEX) for secondary indexes.
        // - Must use the full column definition even for updates.
        $sql = "CREATE TABLE {$table_name} (
            id bigint(20) unsigned NOT NULL AUTO_INCREMENT,
            user_id bigint(20) unsigned NOT NULL DEFAULT 0,
            title varchar(255) NOT NULL DEFAULT '',
            content longtext NOT NULL DEFAULT '',
            status varchar(20) NOT NULL DEFAULT 'active',
            sort_order int(11) NOT NULL DEFAULT 0,
            created_at datetime NOT NULL DEFAULT CURRENT_TIMESTAMP,
            updated_at datetime NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
            PRIMARY KEY  (id),
            KEY user_id (user_id),
            KEY status (status)
        ) {$charset_collate};";

        require_once ABSPATH . 'wp-admin/includes/upgrade.php';
        dbDelta( $sql );

        update_option( 'plugin_name_db_version', '1.0.0' );
    }

    private static function set_default_options(): void {
        $defaults = [
            'api_key'         => '',
            'items_per_page'  => 10,
            'enable_caching'  => true,
            'cache_ttl'       => 3600,
        ];

        // Only set if the option doesn't already exist (preserves settings on reactivation).
        if ( false === get_option( 'plugin_name_settings' ) ) {
            add_option( 'plugin_name_settings', $defaults );
        }
    }
}
```

## Deactivator

Runs when the plugin is deactivated. Should be non-destructive — the user might reactivate.

```php
<?php

declare(strict_types=1);

namespace PluginName;

class Deactivator {

    public static function deactivate(): void {
        // Clear all scheduled cron events.
        wp_clear_scheduled_hook( 'plugin_name_daily_sync' );
        wp_clear_scheduled_hook( 'plugin_name_hourly_cleanup' );

        // Flush rewrite rules to remove custom post type URLs.
        flush_rewrite_rules();

        // Clear transients (optional — they expire naturally).
        delete_transient( 'plugin_name_cache' );

        // Do NOT delete options, tables, or user data here.
        // That belongs in uninstall.php.
    }
}
```

## Uninstall (uninstall.php)

Runs when the plugin is deleted via the admin. This is the permanent cleanup.

```php
<?php
/**
 * Uninstall handler for Plugin Name.
 *
 * Runs when the plugin is deleted via the WordPress admin.
 * Removes all plugin data from the database.
 */

// Prevent direct access.
if ( ! defined( 'WP_UNINSTALL_PLUGIN' ) ) {
    exit;
}

global $wpdb;

// Remove plugin options.
delete_option( 'plugin_name_settings' );
delete_option( 'plugin_name_db_version' );

// Remove all options matching the plugin prefix (for multisite or dynamic options).
$wpdb->query(
    $wpdb->prepare(
        "DELETE FROM {$wpdb->options} WHERE option_name LIKE %s",
        $wpdb->esc_like( 'plugin_name_' ) . '%'
    )
);

// Drop custom tables.
$wpdb->query( "DROP TABLE IF EXISTS {$wpdb->prefix}plugin_name_items" );

// Remove post meta from all posts.
delete_metadata( 'post', 0, '_plugin_name_status', '', true );
delete_metadata( 'post', 0, '_plugin_name_data', '', true );

// Remove user meta from all users.
delete_metadata( 'user', 0, '_plugin_name_preferences', '', true );

// Remove transients.
$wpdb->query(
    $wpdb->prepare(
        "DELETE FROM {$wpdb->options} WHERE option_name LIKE %s OR option_name LIKE %s",
        $wpdb->esc_like( '_transient_plugin_name_' ) . '%',
        $wpdb->esc_like( '_transient_timeout_plugin_name_' ) . '%'
    )
);

// Remove custom posts and their meta.
$posts = get_posts( [
    'post_type'      => 'plugin_name_item',
    'post_status'    => 'any',
    'posts_per_page' => -1,
    'fields'         => 'ids',
] );

foreach ( $posts as $post_id ) {
    wp_delete_post( $post_id, true );
}

// Remove custom taxonomies' terms.
$terms = get_terms( [
    'taxonomy'   => 'plugin_name_category',
    'hide_empty' => false,
    'fields'     => 'ids',
] );

if ( ! is_wp_error( $terms ) ) {
    foreach ( $terms as $term_id ) {
        wp_delete_term( $term_id, 'plugin_name_category' );
    }
}

// Remove custom capabilities from roles.
$role = get_role( 'administrator' );
if ( $role ) {
    $role->remove_cap( 'plugin_name_manage_items' );
}

// Clear any remaining cron events.
wp_clear_scheduled_hook( 'plugin_name_daily_sync' );
wp_clear_scheduled_hook( 'plugin_name_hourly_cleanup' );
```

## Database Migrations

Check the stored DB version on `plugins_loaded` and run migrations when needed:

```php
add_action( 'plugins_loaded', static function (): void {
    $current_version = get_option( 'plugin_name_db_version', '0.0.0' );

    if ( version_compare( $current_version, '1.1.0', '<' ) ) {
        self::migrate_to_1_1_0();
        update_option( 'plugin_name_db_version', '1.1.0' );
    }

    if ( version_compare( $current_version, '1.2.0', '<' ) ) {
        self::migrate_to_1_2_0();
        update_option( 'plugin_name_db_version', '1.2.0' );
    }
} );

private static function migrate_to_1_1_0(): void {
    global $wpdb;

    $table_name = $wpdb->prefix . 'plugin_name_items';

    // Add a new column — dbDelta handles this if you rerun the full CREATE TABLE.
    // Or do it manually for simple additions:
    $column_exists = $wpdb->get_results(
        $wpdb->prepare(
            "SHOW COLUMNS FROM {$table_name} LIKE %s",
            'priority'
        )
    );

    if ( empty( $column_exists ) ) {
        $wpdb->query( "ALTER TABLE {$table_name} ADD COLUMN priority int(11) NOT NULL DEFAULT 0 AFTER sort_order" );
    }
}
```

## Cron Scheduling

```php
// Schedule events on activation.
public static function activate(): void {
    // Schedule daily sync at midnight.
    if ( ! wp_next_scheduled( 'plugin_name_daily_sync' ) ) {
        wp_schedule_event( time(), 'daily', 'plugin_name_daily_sync' );
    }

    // Schedule hourly cleanup.
    if ( ! wp_next_scheduled( 'plugin_name_hourly_cleanup' ) ) {
        wp_schedule_event( time(), 'hourly', 'plugin_name_hourly_cleanup' );
    }
}

// Register custom intervals if needed.
add_filter( 'cron_schedules', static function ( array $schedules ): array {
    $schedules['plugin_name_every_5_minutes'] = [
        'interval' => 300,
        'display'  => __( 'Every 5 Minutes', 'plugin-name' ),
    ];
    return $schedules;
} );

// Handle the events.
add_action( 'plugin_name_daily_sync', static function (): void {
    // Run the daily sync.
    $service = new \PluginName\Services\SyncService();
    $service->run();
} );

add_action( 'plugin_name_hourly_cleanup', static function (): void {
    // Clean up expired items.
    global $wpdb;
    $wpdb->query(
        $wpdb->prepare(
            "DELETE FROM {$wpdb->prefix}plugin_name_items WHERE status = %s AND updated_at < %s",
            'expired',
            gmdate( 'Y-m-d H:i:s', strtotime( '-30 days' ) )
        )
    );
} );
```

Always unschedule cron events in the Deactivator with `wp_clear_scheduled_hook()`.

## WP-CLI Commands

```php
if ( defined( 'WP_CLI' ) && WP_CLI ) {
    \WP_CLI::add_command( 'plugin-name', \PluginName\CLI\Plugin_Command::class );
}
```

```php
<?php

declare(strict_types=1);

namespace PluginName\CLI;

use WP_CLI;

class Plugin_Command {

    /**
     * Syncs items from the external API.
     *
     * ## OPTIONS
     *
     * [--dry-run]
     * : Preview changes without applying them.
     *
     * [--limit=<number>]
     * : Maximum items to sync. Default: all.
     *
     * ## EXAMPLES
     *
     *     wp plugin-name sync
     *     wp plugin-name sync --dry-run
     *     wp plugin-name sync --limit=50
     *
     * @param array $args       Positional arguments.
     * @param array $assoc_args Named arguments.
     */
    public function sync( array $args, array $assoc_args ): void {
        $dry_run = \WP_CLI\Utils\get_flag_value( $assoc_args, 'dry-run', false );
        $limit   = (int) ( $assoc_args['limit'] ?? 0 );

        if ( $dry_run ) {
            WP_CLI::log( 'Dry run mode — no changes will be made.' );
        }

        $service = new \PluginName\Services\SyncService();
        $items   = $service->fetch( $limit ?: null );

        $progress = \WP_CLI\Utils\make_progress_bar( 'Syncing items', count( $items ) );

        foreach ( $items as $item ) {
            if ( ! $dry_run ) {
                $service->save( $item );
            }
            $progress->tick();
        }

        $progress->finish();

        WP_CLI::success(
            sprintf( '%d items %s.', count( $items ), $dry_run ? 'found' : 'synced' )
        );
    }

    /**
     * Shows plugin status and diagnostics.
     *
     * ## EXAMPLES
     *
     *     wp plugin-name status
     */
    public function status(): void {
        global $wpdb;

        $db_version = get_option( 'plugin_name_db_version', 'not set' );
        $item_count = (int) $wpdb->get_var(
            "SELECT COUNT(*) FROM {$wpdb->prefix}plugin_name_items"
        );

        $next_sync = wp_next_scheduled( 'plugin_name_daily_sync' );

        WP_CLI::log( "Plugin version: " . PLUGIN_NAME_VERSION );
        WP_CLI::log( "DB version: {$db_version}" );
        WP_CLI::log( "Total items: {$item_count}" );
        WP_CLI::log( "Next sync: " . ( $next_sync ? gmdate( 'Y-m-d H:i:s', $next_sync ) : 'not scheduled' ) );
    }

    /**
     * Resets the plugin to default state.
     *
     * ## OPTIONS
     *
     * [--yes]
     * : Skip confirmation prompt.
     *
     * ## EXAMPLES
     *
     *     wp plugin-name reset --yes
     */
    public function reset( array $args, array $assoc_args ): void {
        WP_CLI::confirm( 'This will delete all plugin data. Continue?', $assoc_args );

        global $wpdb;
        $wpdb->query( "TRUNCATE TABLE {$wpdb->prefix}plugin_name_items" );
        delete_option( 'plugin_name_settings' );

        \PluginName\Activator::activate();

        WP_CLI::success( 'Plugin reset to defaults.' );
    }
}
```
