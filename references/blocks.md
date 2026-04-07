# Blocks Reference

Full templates for custom Gutenberg blocks including block.json, editor components, server-side rendering, and the Interactivity API.

## block.json Template

Every block must have a `block.json` file. This is the canonical way to register blocks.

```json
{
    "$schema": "https://schemas.wp.org/trunk/block.json",
    "apiVersion": 3,
    "name": "plugin-name/example-block",
    "version": "1.0.0",
    "title": "Example Block",
    "category": "widgets",
    "icon": "smiley",
    "description": "An example block for demonstration.",
    "keywords": [ "example", "demo" ],
    "supports": {
        "html": false,
        "color": {
            "background": true,
            "text": true,
            "link": true
        },
        "spacing": {
            "margin": true,
            "padding": true,
            "blockGap": true
        },
        "typography": {
            "fontSize": true,
            "lineHeight": true
        },
        "align": [ "wide", "full" ]
    },
    "attributes": {
        "heading": {
            "type": "string",
            "default": ""
        },
        "count": {
            "type": "number",
            "default": 3
        },
        "showDate": {
            "type": "boolean",
            "default": true
        }
    },
    "textdomain": "plugin-name",
    "editorScript": "file:./index.js",
    "editorStyle": "file:./index.css",
    "style": "file:./style-index.css",
    "render": "file:./render.php",
    "viewScript": "file:./view.js"
}
```

## index.js (Entry Point)

```js
import { registerBlockType } from '@wordpress/blocks';
import Edit from './edit';
import save from './save';
import metadata from './block.json';
import './style.scss';
import './editor.scss';

registerBlockType( metadata.name, {
    edit: Edit,
    save,
} );
```

## edit.js (Editor Component)

```jsx
import { __ } from '@wordpress/i18n';
import {
    useBlockProps,
    InspectorControls,
    RichText,
} from '@wordpress/block-editor';
import {
    PanelBody,
    RangeControl,
    ToggleControl,
} from '@wordpress/components';

export default function Edit( { attributes, setAttributes } ) {
    const { heading, count, showDate } = attributes;
    const blockProps = useBlockProps();

    return (
        <>
            <InspectorControls>
                <PanelBody title={ __( 'Settings', 'plugin-name' ) }>
                    <RangeControl
                        label={ __( 'Number of items', 'plugin-name' ) }
                        value={ count }
                        onChange={ ( value ) => setAttributes( { count: value } ) }
                        min={ 1 }
                        max={ 10 }
                    />
                    <ToggleControl
                        label={ __( 'Show date', 'plugin-name' ) }
                        checked={ showDate }
                        onChange={ ( value ) => setAttributes( { showDate: value } ) }
                    />
                </PanelBody>
            </InspectorControls>
            <div { ...blockProps }>
                <RichText
                    tagName="h2"
                    value={ heading }
                    onChange={ ( value ) => setAttributes( { heading: value } ) }
                    placeholder={ __( 'Enter heading…', 'plugin-name' ) }
                />
                <p>{ __( 'Block preview content here.', 'plugin-name' ) }</p>
            </div>
        </>
    );
}
```

## save.js (Static Blocks)

For blocks with static content saved to the database:

```jsx
import { useBlockProps, RichText } from '@wordpress/block-editor';

export default function save( { attributes } ) {
    const { heading } = attributes;
    const blockProps = useBlockProps.save();

    return (
        <div { ...blockProps }>
            <RichText.Content tagName="h2" value={ heading } />
        </div>
    );
}
```

For **dynamic blocks** (server-rendered), return `null` from save:

```jsx
export default function save() {
    return null;
}
```

## render.php (Dynamic Block Rendering)

For dynamic blocks, use `render.php` referenced in block.json. This file receives `$attributes`, `$content`, and `$block` variables automatically.

```php
<?php
/**
 * Render callback for the example block.
 *
 * @param array    $attributes Block attributes.
 * @param string   $content    Block content.
 * @param WP_Block $block      Block instance.
 */

declare(strict_types=1);

$heading   = $attributes['heading'] ?? '';
$count     = $attributes['count'] ?? 3;
$show_date = $attributes['showDate'] ?? true;

$query = new WP_Query( [
    'post_type'      => 'post',
    'posts_per_page' => $count,
    'post_status'    => 'publish',
] );

if ( ! $query->have_posts() ) {
    return;
}
?>
<div <?php echo get_block_wrapper_attributes(); ?>>
    <?php if ( $heading ) : ?>
        <h2><?php echo wp_kses_post( $heading ); ?></h2>
    <?php endif; ?>

    <ul>
        <?php while ( $query->have_posts() ) : ?>
            <?php $query->the_post(); ?>
            <li>
                <a href="<?php the_permalink(); ?>"><?php the_title(); ?></a>
                <?php if ( $show_date ) : ?>
                    <time datetime="<?php echo esc_attr( get_the_date( 'c' ) ); ?>">
                        <?php echo esc_html( get_the_date() ); ?>
                    </time>
                <?php endif; ?>
            </li>
        <?php endwhile; ?>
    </ul>
</div>
<?php
wp_reset_postdata();
```

## Server-Side Block Registration

```php
add_action( 'init', static function (): void {
    // Register all blocks from the build directory.
    // Each subdirectory with a block.json gets registered.
    $blocks = glob( PLUGIN_NAME_DIR . 'build/*/block.json' );

    foreach ( $blocks as $block_json ) {
        register_block_type( dirname( $block_json ) );
    }
} );
```

Or register a single block:

```php
register_block_type( PLUGIN_NAME_DIR . 'build/example-block' );
```

## Interactivity API

For blocks that need client-side interactivity without a full React app. Uses server-rendered HTML with declarative directives.

### render.php with Interactivity API

```php
<?php
declare(strict_types=1);

wp_interactivity_state( 'plugin-name/accordion', [
    'isOpen' => false,
] );
?>
<div
    <?php echo get_block_wrapper_attributes(); ?>
    data-wp-interactive="plugin-name/accordion"
>
    <button
        data-wp-on--click="actions.toggle"
        data-wp-bind--aria-expanded="state.isOpen"
        aria-controls="accordion-content"
    >
        <?php echo esc_html( $attributes['heading'] ?? __( 'Toggle', 'plugin-name' ) ); ?>
        <span data-wp-text="state.isOpen ? '▲' : '▼'"></span>
    </button>

    <div
        id="accordion-content"
        data-wp-bind--hidden="!state.isOpen"
        data-wp-class--is-visible="state.isOpen"
    >
        <?php echo wp_kses_post( $content ); ?>
    </div>
</div>
```

### view.js (Interactivity Store)

```js
import { store, getContext } from '@wordpress/interactivity';

store( 'plugin-name/accordion', {
    actions: {
        toggle() {
            const state = store( 'plugin-name/accordion' ).state;
            state.isOpen = ! state.isOpen;
        },
    },
} );
```

### block.json additions for Interactivity API

Add these to block.json:

```json
{
    "supports": {
        "interactivity": true
    },
    "viewScriptModule": "file:./view.js"
}
```

Note: Use `viewScriptModule` (not `viewScript`) for Interactivity API blocks — it loads as an ES module.

### Common Directives

| Directive | Purpose |
|-----------|---------|
| `data-wp-interactive="namespace"` | Marks the interactive region |
| `data-wp-on--click="actions.fn"` | Event handler |
| `data-wp-bind--attr="state.val"` | Bind attribute to state |
| `data-wp-text="state.val"` | Set text content |
| `data-wp-class--name="state.bool"` | Toggle CSS class |
| `data-wp-bind--hidden="!state.bool"` | Show/hide element |
| `data-wp-context='{"key":"val"}'` | Local context for subtree |
| `data-wp-each="state.items"` | Loop over array |
| `data-wp-init="callbacks.onInit"` | Run on mount |
| `data-wp-watch="callbacks.onChange"` | React to state changes |

## Block Patterns

Register block patterns to provide pre-configured block combinations:

```php
add_action( 'init', static function (): void {
    register_block_pattern(
        'plugin-name/hero-section',
        [
            'title'       => __( 'Hero Section', 'plugin-name' ),
            'description' => __( 'A hero section with heading and CTA.', 'plugin-name' ),
            'categories'  => [ 'featured' ],
            'content'     => '<!-- wp:group {"align":"full","layout":{"type":"constrained"}} -->
                <div class="wp-block-group alignfull">
                    <!-- wp:heading {"level":1} -->
                    <h1 class="wp-block-heading">Your Heading Here</h1>
                    <!-- /wp:heading -->
                    <!-- wp:buttons -->
                    <div class="wp-block-buttons">
                        <!-- wp:button -->
                        <div class="wp-block-button"><a class="wp-block-button__link">Get Started</a></div>
                        <!-- /wp:button -->
                    </div>
                    <!-- /wp:buttons -->
                </div>
                <!-- /wp:group -->',
        ]
    );
} );
```

## Block Styles

Register visual variations for existing blocks:

```php
add_action( 'init', static function (): void {
    register_block_style( 'core/button', [
        'name'  => 'plugin-name-outline',
        'label' => __( 'Outline', 'plugin-name' ),
    ] );
} );
```
