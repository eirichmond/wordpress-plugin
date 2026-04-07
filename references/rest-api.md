# REST API Reference

Full WP_REST_Controller implementation with CRUD routes, permissions, schema, and registration.

## Controller Example

```php
<?php

declare(strict_types=1);

namespace PluginName\REST;

use WP_REST_Controller;
use WP_REST_Request;
use WP_REST_Response;
use WP_REST_Server;
use WP_Error;

class Items_Controller extends WP_REST_Controller {

    public function __construct() {
        $this->namespace = 'plugin-name/v1';
        $this->rest_base = 'items';
    }

    /**
     * Register routes.
     */
    public function register_routes(): void {
        // Collection route: GET (list) and POST (create).
        register_rest_route(
            $this->namespace,
            '/' . $this->rest_base,
            [
                [
                    'methods'             => WP_REST_Server::READABLE,
                    'callback'            => [ $this, 'get_items' ],
                    'permission_callback' => [ $this, 'get_items_permissions_check' ],
                    'args'                => $this->get_collection_params(),
                ],
                [
                    'methods'             => WP_REST_Server::CREATABLE,
                    'callback'            => [ $this, 'create_item' ],
                    'permission_callback' => [ $this, 'create_item_permissions_check' ],
                    'args'                => $this->get_endpoint_args_for_item_schema( WP_REST_Server::CREATABLE ),
                ],
                'schema' => [ $this, 'get_public_item_schema' ],
            ]
        );

        // Single item route: GET, PUT/PATCH, DELETE.
        register_rest_route(
            $this->namespace,
            '/' . $this->rest_base . '/(?P<id>[\d]+)',
            [
                'args' => [
                    'id' => [
                        'description' => __( 'Unique identifier for the item.', 'plugin-name' ),
                        'type'        => 'integer',
                    ],
                ],
                [
                    'methods'             => WP_REST_Server::READABLE,
                    'callback'            => [ $this, 'get_item' ],
                    'permission_callback' => [ $this, 'get_item_permissions_check' ],
                ],
                [
                    'methods'             => WP_REST_Server::EDITABLE,
                    'callback'            => [ $this, 'update_item' ],
                    'permission_callback' => [ $this, 'update_item_permissions_check' ],
                    'args'                => $this->get_endpoint_args_for_item_schema( WP_REST_Server::EDITABLE ),
                ],
                [
                    'methods'             => WP_REST_Server::DELETABLE,
                    'callback'            => [ $this, 'delete_item' ],
                    'permission_callback' => [ $this, 'delete_item_permissions_check' ],
                ],
                'schema' => [ $this, 'get_public_item_schema' ],
            ]
        );
    }

    /**
     * Permissions: always return true or WP_Error, never false.
     */
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

    public function create_item_permissions_check( WP_REST_Request $request ): true|WP_Error {
        if ( ! current_user_can( 'publish_posts' ) ) {
            return new WP_Error(
                'rest_forbidden',
                __( 'You do not have permission to create items.', 'plugin-name' ),
                [ 'status' => rest_authorization_required_code() ]
            );
        }
        return true;
    }

    public function get_item_permissions_check( WP_REST_Request $request ): true|WP_Error {
        return $this->get_items_permissions_check( $request );
    }

    public function update_item_permissions_check( WP_REST_Request $request ): true|WP_Error {
        if ( ! current_user_can( 'edit_posts' ) ) {
            return new WP_Error(
                'rest_forbidden',
                __( 'You do not have permission to update items.', 'plugin-name' ),
                [ 'status' => rest_authorization_required_code() ]
            );
        }
        return true;
    }

    public function delete_item_permissions_check( WP_REST_Request $request ): true|WP_Error {
        if ( ! current_user_can( 'delete_posts' ) ) {
            return new WP_Error(
                'rest_forbidden',
                __( 'You do not have permission to delete items.', 'plugin-name' ),
                [ 'status' => rest_authorization_required_code() ]
            );
        }
        return true;
    }

    /**
     * GET /items — list items.
     */
    public function get_items( WP_REST_Request $request ): WP_REST_Response {
        $per_page = $request->get_param( 'per_page' ) ?? 10;
        $page     = $request->get_param( 'page' ) ?? 1;

        $query = new \WP_Query( [
            'post_type'      => 'plugin_name_item',
            'posts_per_page' => $per_page,
            'paged'          => $page,
            'post_status'    => 'publish',
        ] );

        $items = [];
        foreach ( $query->posts as $post ) {
            $items[] = $this->prepare_item_for_response( $post, $request )->get_data();
        }

        $response = new WP_REST_Response( $items, 200 );

        // Pagination headers.
        $response->header( 'X-WP-Total', (string) $query->found_posts );
        $response->header( 'X-WP-TotalPages', (string) $query->max_num_pages );

        return $response;
    }

    /**
     * GET /items/{id} — single item.
     */
    public function get_item( WP_REST_Request $request ): WP_REST_Response|WP_Error {
        $post = get_post( $request->get_param( 'id' ) );

        if ( ! $post || 'plugin_name_item' !== $post->post_type ) {
            return new WP_Error(
                'rest_not_found',
                __( 'Item not found.', 'plugin-name' ),
                [ 'status' => 404 ]
            );
        }

        return $this->prepare_item_for_response( $post, $request );
    }

    /**
     * POST /items — create item.
     */
    public function create_item( WP_REST_Request $request ): WP_REST_Response|WP_Error {
        $post_id = wp_insert_post( [
            'post_type'    => 'plugin_name_item',
            'post_title'   => sanitize_text_field( $request->get_param( 'title' ) ),
            'post_content' => wp_kses_post( $request->get_param( 'content' ) ?? '' ),
            'post_status'  => 'publish',
        ], true );

        if ( is_wp_error( $post_id ) ) {
            return $post_id;
        }

        // Set meta.
        if ( $request->get_param( 'status' ) ) {
            update_post_meta( $post_id, '_plugin_name_status', sanitize_text_field( $request->get_param( 'status' ) ) );
        }

        $post     = get_post( $post_id );
        $response = $this->prepare_item_for_response( $post, $request );
        $response->set_status( 201 );
        $response->header( 'Location', rest_url( "{$this->namespace}/{$this->rest_base}/{$post_id}" ) );

        return $response;
    }

    /**
     * PUT/PATCH /items/{id} — update item.
     */
    public function update_item( WP_REST_Request $request ): WP_REST_Response|WP_Error {
        $post = get_post( $request->get_param( 'id' ) );

        if ( ! $post || 'plugin_name_item' !== $post->post_type ) {
            return new WP_Error( 'rest_not_found', __( 'Item not found.', 'plugin-name' ), [ 'status' => 404 ] );
        }

        $args = [ 'ID' => $post->ID ];

        if ( $request->get_param( 'title' ) !== null ) {
            $args['post_title'] = sanitize_text_field( $request->get_param( 'title' ) );
        }

        if ( $request->get_param( 'content' ) !== null ) {
            $args['post_content'] = wp_kses_post( $request->get_param( 'content' ) );
        }

        $updated = wp_update_post( $args, true );

        if ( is_wp_error( $updated ) ) {
            return $updated;
        }

        return $this->prepare_item_for_response( get_post( $post->ID ), $request );
    }

    /**
     * DELETE /items/{id} — delete item.
     */
    public function delete_item( WP_REST_Request $request ): WP_REST_Response|WP_Error {
        $post = get_post( $request->get_param( 'id' ) );

        if ( ! $post || 'plugin_name_item' !== $post->post_type ) {
            return new WP_Error( 'rest_not_found', __( 'Item not found.', 'plugin-name' ), [ 'status' => 404 ] );
        }

        $deleted = wp_delete_post( $post->ID, true );

        if ( ! $deleted ) {
            return new WP_Error( 'rest_cannot_delete', __( 'Failed to delete item.', 'plugin-name' ), [ 'status' => 500 ] );
        }

        return new WP_REST_Response( null, 204 );
    }

    /**
     * Prepare a post for the response.
     */
    public function prepare_item_for_response( $post, WP_REST_Request $request ): WP_REST_Response {
        $data = [
            'id'      => $post->ID,
            'title'   => $post->post_title,
            'content' => $post->post_content,
            'status'  => get_post_meta( $post->ID, '_plugin_name_status', true ) ?: 'active',
            'date'    => mysql_to_rfc3339( $post->post_date ),
        ];

        return new WP_REST_Response( $data, 200 );
    }

    /**
     * Schema for items.
     */
    public function get_item_schema(): array {
        return [
            '$schema'    => 'http://json-schema.org/draft-04/schema#',
            'title'      => 'plugin-name-item',
            'type'       => 'object',
            'properties' => [
                'id'      => [
                    'description' => __( 'Unique identifier.', 'plugin-name' ),
                    'type'        => 'integer',
                    'context'     => [ 'view', 'edit' ],
                    'readonly'    => true,
                ],
                'title'   => [
                    'description' => __( 'Item title.', 'plugin-name' ),
                    'type'        => 'string',
                    'context'     => [ 'view', 'edit' ],
                    'required'    => true,
                ],
                'content' => [
                    'description' => __( 'Item content.', 'plugin-name' ),
                    'type'        => 'string',
                    'context'     => [ 'view', 'edit' ],
                ],
                'status'  => [
                    'description' => __( 'Item status.', 'plugin-name' ),
                    'type'        => 'string',
                    'enum'        => [ 'active', 'inactive', 'archived' ],
                    'context'     => [ 'view', 'edit' ],
                ],
                'date'    => [
                    'description' => __( 'Created date (RFC3339).', 'plugin-name' ),
                    'type'        => 'string',
                    'format'      => 'date-time',
                    'context'     => [ 'view' ],
                    'readonly'    => true,
                ],
            ],
        ];
    }
}
```

## Registration

```php
add_action( 'rest_api_init', static function (): void {
    ( new \PluginName\REST\Items_Controller() )->register_routes();
} );
```

## JavaScript Usage

```js
import apiFetch from '@wordpress/api-fetch';

// GET items.
const items = await apiFetch( { path: '/plugin-name/v1/items' } );

// POST new item.
const newItem = await apiFetch( {
    path: '/plugin-name/v1/items',
    method: 'POST',
    data: { title: 'New Item', content: 'Content here.' },
} );

// DELETE item.
await apiFetch( {
    path: `/plugin-name/v1/items/${ itemId }`,
    method: 'DELETE',
} );
```

`@wordpress/api-fetch` handles nonces and base URL automatically when used within the WordPress admin.
