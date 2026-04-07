# Testing Reference

PHPUnit setup, wp-env configuration, WP_UnitTestCase examples, and Playwright end-to-end tests.

## wp-env Configuration (.wp-env.json)

```json
{
    "core": null,
    "phpVersion": "8.2",
    "plugins": [ "." ],
    "mappings": {
        "wp-content/mu-plugins/test-utils.php": "./tests/php/test-utils.php"
    },
    "env": {
        "tests": {
            "phpVersion": "8.2"
        }
    }
}
```

Start the environment:

```bash
npx wp-env start
```

## PHPUnit Configuration (phpunit.xml.dist)

```xml
<?xml version="1.0"?>
<phpunit
    bootstrap="tests/php/bootstrap.php"
    backupGlobals="false"
    colors="true"
    convertErrorsToExceptions="true"
    convertNoticesToExceptions="true"
    convertWarningsToExceptions="true"
>
    <testsuites>
        <testsuite name="unit">
            <directory suffix="Test.php">./tests/php/Unit</directory>
        </testsuite>
        <testsuite name="integration">
            <directory suffix="Test.php">./tests/php/Integration</directory>
        </testsuite>
    </testsuites>
    <coverage>
        <include>
            <directory suffix=".php">./src</directory>
        </include>
    </coverage>
</phpunit>
```

## Test Bootstrap (tests/php/bootstrap.php)

```php
<?php

declare(strict_types=1);

// Load Composer autoloader for plugin classes.
require_once dirname( __DIR__, 2 ) . '/vendor/autoload.php';

// Determine if we're running unit tests (no WP) or integration tests (with WP).
$suite = getenv( 'WP_TESTS_SUITE' ) ?: 'integration';

if ( 'unit' === $suite ) {
    // Unit tests don't need WordPress.
    return;
}

// Integration tests — load WordPress test framework.
$_tests_dir = getenv( 'WP_TESTS_DIR' ) ?: '/tmp/wordpress-tests-lib';

if ( ! file_exists( "{$_tests_dir}/includes/functions.php" ) ) {
    echo "Could not find WordPress test framework at {$_tests_dir}.\n";
    echo "Use wp-env: npx wp-env run tests-cli --env-cwd=wp-content/plugins/plugin-name vendor/bin/phpunit\n";
    exit( 1 );
}

// Load the plugin before WordPress initialises.
tests_add_filter( 'muplugins_loaded', static function (): void {
    require dirname( __DIR__, 2 ) . '/plugin-name.php';
} );

require "{$_tests_dir}/includes/bootstrap.php";
```

## Running Tests

```bash
# Integration tests via wp-env.
npx wp-env run tests-cli --env-cwd=wp-content/plugins/plugin-name vendor/bin/phpunit

# Unit tests only (no WordPress needed).
WP_TESTS_SUITE=unit vendor/bin/phpunit --testsuite=unit

# With coverage.
npx wp-env run tests-cli --env-cwd=wp-content/plugins/plugin-name vendor/bin/phpunit --coverage-html=coverage
```

## Unit Test Example (No WordPress)

Unit tests are for pure business logic with no WordPress dependency. Mock WordPress functions if needed.

```php
<?php

declare(strict_types=1);

namespace PluginName\Tests\Unit;

use PHPUnit\Framework\TestCase;
use PluginName\Services\PriceCalculator;

class PriceCalculatorTest extends TestCase {

    private PriceCalculator $calculator;

    protected function setUp(): void {
        $this->calculator = new PriceCalculator();
    }

    public function test_calculates_total_with_tax(): void {
        $result = $this->calculator->calculate( 100.00, 0.20 );

        $this->assertSame( 120.00, $result );
    }

    public function test_applies_discount_before_tax(): void {
        $result = $this->calculator->calculate( 100.00, 0.20, discount: 10.00 );

        $this->assertSame( 108.00, $result );
    }

    public function test_rejects_negative_price(): void {
        $this->expectException( \InvalidArgumentException::class );

        $this->calculator->calculate( -50.00, 0.20 );
    }

    /**
     * @dataProvider priceProvider
     */
    public function test_various_prices( float $price, float $tax, float $expected ): void {
        $this->assertSame( $expected, $this->calculator->calculate( $price, $tax ) );
    }

    public static function priceProvider(): array {
        return [
            'zero price' => [ 0.00, 0.20, 0.00 ],
            'zero tax'   => [ 100.00, 0.00, 100.00 ],
            'standard'   => [ 50.00, 0.10, 55.00 ],
        ];
    }
}
```

## Integration Test Example (With WordPress)

Integration tests use `WP_UnitTestCase` and have full access to WordPress.

```php
<?php

declare(strict_types=1);

namespace PluginName\Tests\Integration;

use WP_UnitTestCase;

class PostType_Test extends WP_UnitTestCase {

    public function test_post_type_is_registered(): void {
        $this->assertTrue( post_type_exists( 'plugin_name_item' ) );
    }

    public function test_post_type_supports_editor(): void {
        $this->assertTrue( post_type_supports( 'plugin_name_item', 'editor' ) );
    }

    public function test_post_type_is_available_in_rest(): void {
        $post_type = get_post_type_object( 'plugin_name_item' );

        $this->assertTrue( $post_type->show_in_rest );
    }

    public function test_creating_item_sets_default_meta(): void {
        $post_id = self::factory()->post->create( [
            'post_type' => 'plugin_name_item',
        ] );

        // Trigger the save_post hook.
        do_action( 'save_post_plugin_name_item', $post_id, get_post( $post_id ), false );

        $this->assertSame(
            'draft',
            get_post_meta( $post_id, '_plugin_name_status', true )
        );
    }
}
```

## REST API Integration Test

```php
<?php

declare(strict_types=1);

namespace PluginName\Tests\Integration;

use WP_REST_Request;
use WP_UnitTestCase;

class REST_Items_Test extends WP_UnitTestCase {

    private int $admin_id;

    public function set_up(): void {
        parent::set_up();

        $this->admin_id = self::factory()->user->create( [ 'role' => 'administrator' ] );
    }

    public function test_get_items_returns_200(): void {
        wp_set_current_user( $this->admin_id );

        $request  = new WP_REST_Request( 'GET', '/plugin-name/v1/items' );
        $response = rest_do_request( $request );

        $this->assertSame( 200, $response->get_status() );
    }

    public function test_get_items_requires_authentication(): void {
        wp_set_current_user( 0 );

        $request  = new WP_REST_Request( 'GET', '/plugin-name/v1/items' );
        $response = rest_do_request( $request );

        $this->assertSame( 401, $response->get_status() );
    }

    public function test_create_item(): void {
        wp_set_current_user( $this->admin_id );

        $request = new WP_REST_Request( 'POST', '/plugin-name/v1/items' );
        $request->set_body_params( [
            'title'   => 'Test Item',
            'content' => 'Test content.',
        ] );

        $response = rest_do_request( $request );
        $data     = $response->get_data();

        $this->assertSame( 201, $response->get_status() );
        $this->assertSame( 'Test Item', $data['title'] );
    }

    public function test_delete_item(): void {
        wp_set_current_user( $this->admin_id );

        $post_id = self::factory()->post->create( [
            'post_type' => 'plugin_name_item',
        ] );

        $request  = new WP_REST_Request( 'DELETE', "/plugin-name/v1/items/{$post_id}" );
        $response = rest_do_request( $request );

        $this->assertSame( 204, $response->get_status() );
        $this->assertNull( get_post( $post_id ) );
    }
}
```

## Playwright End-to-End Tests

### Configuration (tests/e2e/playwright.config.js)

```js
const { defineConfig } = require( '@playwright/test' );

module.exports = defineConfig( {
    testDir: './specs',
    timeout: 30000,
    use: {
        baseURL: 'http://localhost:8889',
        storageState: './storage-state.json',
    },
    webServer: {
        command: 'npx wp-env start',
        url: 'http://localhost:8889',
        reuseExistingServer: true,
    },
} );
```

### Setup (global-setup.js)

```js
const { request } = require( '@playwright/test' );

module.exports = async () => {
    const requestContext = await request.newContext( {
        baseURL: 'http://localhost:8889',
    } );

    // Log in as admin.
    await requestContext.post( '/wp-login.php', {
        form: {
            log: 'admin',
            pwd: 'password',
            'wp-submit': 'Log In',
            testcookie: '1',
        },
    } );

    // Save storage state.
    await requestContext.storageState( { path: './tests/e2e/storage-state.json' } );
    await requestContext.dispose();
};
```

### Test Spec Example

```js
import { test, expect } from '@wordpress/e2e-test-utils-playwright';

test.describe( 'Plugin Settings Page', () => {

    test( 'settings page loads', async ( { admin, page } ) => {
        await admin.visitAdminPage( 'options-general.php', 'page=plugin-name' );

        await expect( page.locator( 'h1' ) ).toContainText( 'Plugin Name Settings' );
    } );

    test( 'saves settings successfully', async ( { admin, page } ) => {
        await admin.visitAdminPage( 'options-general.php', 'page=plugin-name' );

        await page.fill( '#plugin-name-api-key', 'test-key-123' );
        await page.click( '#submit' );

        await expect( page.locator( '.notice-success' ) ).toBeVisible();
        await expect( page.locator( '#plugin-name-api-key' ) ).toHaveValue( 'test-key-123' );
    } );

    test( 'validates required fields', async ( { admin, page } ) => {
        await admin.visitAdminPage( 'options-general.php', 'page=plugin-name' );

        await page.fill( '#plugin-name-api-key', '' );
        await page.click( '#submit' );

        await expect( page.locator( '.notice-error' ) ).toBeVisible();
    } );
} );

test.describe( 'Custom Block', () => {

    test( 'block can be inserted in editor', async ( { admin, editor, page } ) => {
        await admin.createNewPost();
        await editor.insertBlock( { name: 'plugin-name/example-block' } );

        await expect(
            page.locator( '[data-type="plugin-name/example-block"]' )
        ).toBeVisible();
    } );
} );
```

### Running E2E Tests

```bash
# Start wp-env first.
npx wp-env start

# Run all Playwright tests.
npx wp-scripts test-playwright

# Run specific test file.
npx wp-scripts test-playwright tests/e2e/specs/settings.spec.js

# Run in headed mode for debugging.
npx wp-scripts test-playwright --headed
```
