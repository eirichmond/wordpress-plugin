# Security Reference

Every piece of code that handles user input, output, or capability checks must follow these rules. No exceptions.

## Input Sanitisation

Always sanitise input. Pick the right function for the data type:

| Data Type | Function |
|-----------|----------|
| Plain text | `sanitize_text_field()` |
| Textarea (multiline) | `sanitize_textarea_field()` |
| Email | `sanitize_email()` |
| URL (for storage) | `sanitize_url()` / `esc_url_raw()` |
| Positive integer | `absint()` |
| Integer (any) | `(int) $value` or `intval()` |
| Float | `(float) $value` or `floatval()` |
| HTML (limited tags) | `wp_kses()` / `wp_kses_post()` |
| Filename | `sanitize_file_name()` |
| CSS class | `sanitize_html_class()` |
| Slug | `sanitize_title()` |
| Array of text | `map_deep( $data, 'sanitize_text_field' )` |

## Unslashing Superglobals

WordPress adds slashes to `$_GET`, `$_POST`, `$_REQUEST`, and `$_SERVER`. Always unslash before sanitising:

```php
// Correct — unslash then sanitise.
$name = sanitize_text_field( wp_unslash( $_POST['name'] ?? '' ) );
$email = sanitize_email( wp_unslash( $_POST['email'] ?? '' ) );
$page = absint( $_GET['paged'] ?? 1 );

// Wrong — sanitising without unslashing gives double-escaped data.
$name = sanitize_text_field( $_POST['name'] ?? '' );
```

## Output Escaping

Always escape at the point of echo — not before, not in the variable assignment.

```php
// Text inside HTML elements.
echo '<p>' . esc_html( $user_input ) . '</p>';

// Attribute values.
echo '<input type="text" value="' . esc_attr( $value ) . '">';

// URLs in href/src.
echo '<a href="' . esc_url( $link ) . '">';

// Translated text in HTML.
echo '<h2>' . esc_html__( 'Settings', 'plugin-name' ) . '</h2>';

// Translated text in attributes.
echo '<input placeholder="' . esc_attr__( 'Search...', 'plugin-name' ) . '">';

// HTML you need to allow some tags in.
echo wp_kses_post( $content_with_html );

// Translated string with variable.
printf(
    /* translators: %s: user display name */
    esc_html__( 'Welcome, %s.', 'plugin-name' ),
    esc_html( $user->display_name )
);
```

## Nonce Verification

Every form submission and AJAX request needs a nonce.

### Form Example

```php
// In the form template.
<form method="post" action="">
    <?php wp_nonce_field( 'plugin_name_save_settings', 'plugin_name_nonce' ); ?>
    <input type="text" name="api_key" value="<?php echo esc_attr( $api_key ); ?>">
    <?php submit_button(); ?>
</form>
```

```php
// In the handler.
public function handle_save(): void {
    // Check nonce.
    if (
        ! isset( $_POST['plugin_name_nonce'] )
        || ! wp_verify_nonce(
            sanitize_text_field( wp_unslash( $_POST['plugin_name_nonce'] ) ),
            'plugin_name_save_settings'
        )
    ) {
        wp_die( esc_html__( 'Security check failed.', 'plugin-name' ) );
    }

    // Check capabilities.
    if ( ! current_user_can( 'manage_options' ) ) {
        wp_die( esc_html__( 'You do not have permission to do this.', 'plugin-name' ) );
    }

    // Now safe to process.
    $api_key = sanitize_text_field( wp_unslash( $_POST['api_key'] ?? '' ) );
    update_option( 'plugin_name_api_key', $api_key );

    wp_safe_redirect(
        add_query_arg( 'updated', 'true', wp_get_referer() )
    );
    exit;
}
```

### AJAX Example

```php
// Register the handler.
add_action( 'wp_ajax_plugin_name_delete_item', [ $this, 'ajax_delete_item' ] );

public function ajax_delete_item(): void {
    // Verify nonce.
    check_ajax_referer( 'plugin_name_ajax', 'nonce' );

    // Check capabilities.
    if ( ! current_user_can( 'delete_posts' ) ) {
        wp_send_json_error( [ 'message' => __( 'Permission denied.', 'plugin-name' ) ], 403 );
    }

    $item_id = absint( $_POST['item_id'] ?? 0 );

    if ( 0 === $item_id ) {
        wp_send_json_error( [ 'message' => __( 'Invalid item ID.', 'plugin-name' ) ], 400 );
    }

    // Process deletion.
    $deleted = wp_delete_post( $item_id, true );

    if ( $deleted ) {
        wp_send_json_success( [ 'message' => __( 'Item deleted.', 'plugin-name' ) ] );
    } else {
        wp_send_json_error( [ 'message' => __( 'Failed to delete item.', 'plugin-name' ) ], 500 );
    }
}
```

### Passing the Nonce to JavaScript

```php
wp_localize_script( 'plugin-name-admin', 'pluginNameData', [
    'ajaxUrl' => admin_url( 'admin-ajax.php' ),
    'nonce'   => wp_create_nonce( 'plugin_name_ajax' ),
] );
```

```js
// In JavaScript.
fetch( pluginNameData.ajaxUrl, {
    method: 'POST',
    headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
    body: new URLSearchParams( {
        action: 'plugin_name_delete_item',
        nonce: pluginNameData.nonce,
        item_id: itemId,
    } ),
} );
```

## Capability Checks

Always check the user has permission before doing anything sensitive:

```php
// Admin pages.
if ( ! current_user_can( 'manage_options' ) ) {
    wp_die( esc_html__( 'You do not have permission to access this page.', 'plugin-name' ) );
}

// Post operations — check against the specific post.
if ( ! current_user_can( 'edit_post', $post_id ) ) {
    return;
}

// Custom capabilities.
if ( ! current_user_can( 'plugin_name_manage_items' ) ) {
    return new WP_Error( 'forbidden', __( 'Permission denied.', 'plugin-name' ), [ 'status' => 403 ] );
}
```

## Database Queries

Use `$wpdb->prepare()` for any query with user-supplied values. Never interpolate variables directly.

```php
global $wpdb;

// SELECT with prepare.
$results = $wpdb->get_results(
    $wpdb->prepare(
        "SELECT * FROM {$wpdb->prefix}plugin_name_items WHERE status = %s AND user_id = %d",
        $status,
        $user_id
    )
);

// Single value.
$count = $wpdb->get_var(
    $wpdb->prepare(
        "SELECT COUNT(*) FROM {$wpdb->prefix}plugin_name_items WHERE status = %s",
        'active'
    )
);

// INSERT.
$wpdb->insert(
    "{$wpdb->prefix}plugin_name_items",
    [
        'user_id' => $user_id,
        'title'   => $title,
        'status'  => 'active',
    ],
    [ '%d', '%s', '%s' ]
);

// UPDATE.
$wpdb->update(
    "{$wpdb->prefix}plugin_name_items",
    [ 'status' => 'inactive' ],
    [ 'id' => $item_id ],
    [ '%s' ],
    [ '%d' ]
);

// DELETE.
$wpdb->delete(
    "{$wpdb->prefix}plugin_name_items",
    [ 'id' => $item_id ],
    [ '%d' ]
);
```

Prefer high-level APIs when they cover the use case:

```php
// Use WP_Query instead of raw SQL for posts.
$query = new WP_Query( [
    'post_type'      => 'plugin_name_item',
    'post_status'    => 'publish',
    'posts_per_page' => 10,
    'meta_query'     => [
        [
            'key'   => '_plugin_name_status',
            'value' => 'active',
        ],
    ],
] );

// Use post meta API instead of raw SQL for meta.
update_post_meta( $post_id, '_plugin_name_status', 'active' );
$status = get_post_meta( $post_id, '_plugin_name_status', true );
```

## REST API Permission Callbacks

Always return `true` or `WP_Error` from permission callbacks — never `false` (it produces an unhelpful generic error).

```php
public function get_items_permissions_check( WP_REST_Request $request ): true|WP_Error {
    if ( ! current_user_can( 'read' ) ) {
        return new WP_Error(
            'rest_forbidden',
            __( 'You do not have permission to view items.', 'plugin-name' ),
            [ 'status' => rest_authorization_required_code() ]
        );
    }
    return true;
}
```
