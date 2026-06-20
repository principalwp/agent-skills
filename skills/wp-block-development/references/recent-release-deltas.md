# Recent release deltas (WP 6.7 → 6.9)

Block-development changes a production codebase needs to act on. Each item is a concrete delta — new field, new function, deprecation, or behavior change — not a marketing rundown. Anchored to Make/Core dev notes and Gutenberg release posts.

## Migrations and breaking changes

### `apiVersion: 3` is mandatory for new blocks; lower versions warn (6.9, hard cutover in 7.0)

WP 7.0 will iframe the post editor unconditionally. Any block on `apiVersion` 1 or 2 breaks in 7.0. 6.9 is the deprecation runway — block.json schema validation rejects new blocks below 3, and `SCRIPT_DEBUG` logs warnings for legacy versions.

```json
{ "apiVersion": 3 }
```

Source: [Preparing the post editor for full iframe integration (6.9)](https://make.wordpress.org/core/2025/11/12/preparing-the-post-editor-for-full-iframe-integration/)

### `blockType.parent` must be an array (6.8)

```json
"parent": "core/group"          // warns; may break in future releases
"parent": ["core/group"]         // correct
```

Same applies to `register_block_type()` PHP arg. Easy fix; easy to miss in third-party blocks.

Source: [Misc Block Editor Changes 6.8](https://make.wordpress.org/core/2025/03/25/miscellaneous-block-editor-changes-in-wordpress-6-8/)

### `__experimentalRole` → `role` on attributes (6.7)

Drives contentOnly locking, pattern overrides, and block bindings eligibility. The unstable name was widely used and now logs deprecation warnings.

```js
// before
attributes: { caption: { __experimentalRole: 'content' } }
// after
attributes: { caption: { role: 'content' } }
```

`__experimentalGetBlockAttributesNamesByRole` → `getBlockAttributesNamesByRole`. Valid roles: `content`, `local`. `__experimentalHasContentRoleAttribute` is now private.

### `__experimentalLinkControl*` and friends stabilized (6.8)

Imports of `__experimentalLinkControl`, `__experimentalLinkControlSearchInput`, `__experimentalLinkControlSearchResults`, `__experimentalLinkControlSearchItem` warn on every editor load. Migrate to the stable names. Same release: `getSettings().__unstableIsPreviewMode` → `isPreviewMode`.

### `Navigator` namespace replaces `__experimentalNavigator*` (6.7/6.8); `Navigation` removed in 7.1

Two distinct components — the legacy `Navigation` (block-editor menu UI) is being removed in WP 7.1; the new `Navigator` (router-style screens) is its replacement for inspector UI.

```js
__experimentalNavigatorProvider     → Navigator
__experimentalNavigatorScreen       → Navigator.Screen
__experimentalNavigatorButton       → Navigator.Button
__experimentalNavigatorBackButton   → Navigator.BackButton
goToParent / __experimentalNavigatorToParentButton → goBack / BackButton
```

Also soft-deprecated: `RadioGroup`, `ButtonGroup`. `DimensionControl` is scheduled for WP 7.0 removal.

### Components 36px default deprecated; opt into 40px via `__next40pxDefaultSize` (6.8)

Twenty `@wordpress/components` controls (SelectControl, TextControl, InputControl, RangeControl, NumberControl, FormTokenField, UnitControl, ...) warn at the 36px default. Required for accessibility (touch target). Pass `__next40pxDefaultSize` until the new size becomes default. Modal close button enlarged 24 → 32px; `headerActions` should declare `size="compact"`.

### `core/media` entity deprecated; use `postType/attachment` (6.9)

```js
// deprecated
useEntityProp( 'root', 'media', mediaId, 'caption' )
select( 'core' ).getMedia( mediaId )

// replacement
useEntityRecord( 'postType', 'attachment', mediaId )
select( 'core' ).getEntityRecord( 'postType', 'attachment', mediaId )
```

Plus a silent shape change: `caption` is now `{ raw, rendered }`, not a string. Audit any `caption` consumers.

### `Iframe scale="default"` autoscaling semantics changed (6.8)

For autoscaling previews, switch to `scale="auto-scaled"`. Visible-only-in-snapshots class of regression — custom previews look fine until you take a screenshot.

## New affordances

### Block bindings — full client-side public API (6.7)

6.5 shipped server-only bindings. 6.7 is the first release where you can build editable custom sources end-to-end (read + write + permission gating) from JS.

New `@wordpress/blocks` exports: `registerBlockBindingsSource`, `unregisterBlockBindingsSource`, `getBlockBindingsSource`, `getBlockBindingsSources`. Source registration accepts:

```js
registerBlockBindingsSource( {
    name: 'my/source',
    label: 'My source',
    usesContext: [ 'postId' ],
    getValues:        ( { select, context, bindings } ) => { /* ... */ },
    setValues:        ( { dispatch, context, bindings } ) => { /* unlocks edit mode */ },
    canUserEditValue: ( { select, context, args } ) => true,
    getFieldsList:    ( { select, context } ) => { /* 6.9 — typed dropdown for binding picker */ },
} );
```

`useBlockBindingsUtils()` exposes `updateBlockBindings` and `removeAllBlockBindings`. Server-side companion filter `block_bindings_source_value` ( `$value, $name, $source_args, $block_instance, $attribute_name` ).

Source: [Block Bindings improvements to the editor experience in 6.7](https://make.wordpress.org/core/2024/10/21/block-bindings-improvements-to-the-editor-experience-in-6-7/)

### Block bindings `getFieldsList()` + per-block-type allowlist (6.9)

`getFieldsList()` on the source returns `[{ label, type, args }]` — `type` must match the binding-target attribute's type. `args` is forwarded as `source_args` when the user picks a field. Eliminates the "edit raw block JSON" requirement for non-meta sources.

Companion server filter `block_bindings_supported_attributes_{$block_type}` whitelists which attributes a given block type can bind. Use this to lock down third-party blocks: never expose `style` or `className` to editors.

Source: [Block Bindings improvements in WordPress 6.9](https://make.wordpress.org/core/2025/11/12/block-bindings-improvements-in-wordpress-6-9/)

### post-meta `label` surfaces in the bindings UI (6.7)

Without `label`, the bindings dropdown shows raw meta keys (`_my_plugin_subtitle`), leaking internal naming to editors.

```php
register_post_meta( 'post', 'my_plugin_subtitle', [
    'type'         => 'string',
    'label'        => __( 'Subtitle', 'textdomain' ),
    'show_in_rest' => true,
    'single'       => true,
] );
```

### Block bindings work inside `editor.BlockEdit` filter (Gutenberg 20.0 / WP 6.8)

Custom sources can now layer their UI inside the standard `BlockEdit` higher-order component, unifying with the rest of the InspectorControls toolchain. `addFilter( 'editor.BlockEdit', ... )` receives the bindings-aware edit component; bindings state can be inspected/modified there.

### `block.json` `variations` as PHP file (6.7)

Generates variations dynamically server-side without registering a `variation_callback` in PHP. Cleaner block.json metadata, no separate registration step.

```json
{ "variations": "file:./variations.php" }
```

The file must `return` the array.

### `block.json` top-level `role` field (Gutenberg 21.0 / WP 6.9)

Distinct from per-attribute `role`. Classifies the block type itself for editor surfaces. Shortcode block was the first core consumer.

### `wp_register_block_metadata_collection()` (6.7) + `wp_register_block_types_from_metadata_collection()` (6.8)

Real perf delta in plugins with 20+ blocks: avoids per-file `json_decode()` on every request. Build with `wp-scripts build --blocks-manifest` to emit `blocks-manifest.php`.

```php
if ( function_exists( 'wp_register_block_types_from_metadata_collection' ) ) {
    wp_register_block_types_from_metadata_collection(
        plugin_dir_path( __FILE__ ) . 'build',
        plugin_dir_path( __FILE__ ) . 'build/blocks-manifest.php'
    );
} elseif ( function_exists( 'wp_register_block_metadata_collection' ) ) {
    wp_register_block_metadata_collection(
        plugin_dir_path( __FILE__ ) . 'build',
        plugin_dir_path( __FILE__ ) . 'build/blocks-manifest.php'
    );
    // ... fall back to per-block register_block_type_from_metadata calls
}
```

Compounds across plugins on the same site. Known issue: `wp-scripts start --blocks-manifest` deletes the manifest file in dev mode (April 2025).

Source: [More efficient block type registration in 6.8](https://make.wordpress.org/core/2025/03/13/more-efficient-block-type-registration-in-6-8/)

### `should_load_block_assets_on_demand` filter (6.8)

Decouples on-demand asset loading from `should_load_separate_core_block_assets`. Block themes opt in by default; classic themes can now opt into per-block enqueue without also splitting core stylesheets.

```php
add_filter( 'should_load_block_assets_on_demand', '__return_true' );
```

### Block hooks: `firstChild` / `lastChild` on Template Parts (6.7)

Previously template parts only accepted `before`/`after` siblings — the only way to inject a footer copyright was post-hoc. Now you can hook a block as a child of `core/template-part`.

```json
"blockHooks": { "core/template-part": "lastChild" }
```

### Block hooks now respect `supports.multiple: false` (6.7)

Removes the workaround where every hooked block had to inspect content via `hooked_block_types` to avoid duplicates. Caveat: dedupe is per-context (template, template part, pattern, navigation post) — not per-page. A page composing multiple contexts can still end up with multiple instances.

### Block hooks expanded to post content and synced patterns (Gutenberg 20.0 / WP 6.8)

Hooks were previously template/template-part/navigation only. They now apply to regular post content and synced (reusable) patterns, dramatically widening the surface for plugin-injected blocks. Same `blockHooks` declaration; the registrar's hook-evaluation surface widened internally.

### `setAttributes()` accepts an updater function (6.9)

Eliminates the stale-closure bug class.

```js
// stale-closure trap — `attributes` may be outdated when callback fires
setAttributes( { count: attributes.count + 1 } );

// safe — same shape as React's setState updater
setAttributes( prev => ( { count: prev.count + 1 } ) );
```

### `supports.visibility` — Hide Blocks, default on (6.9)

New default-on supports flag. Hidden blocks are completely omitted from rendered HTML *and* their scripts/styles are not enqueued. **If your block has side effects in `render.php`, those side effects no longer fire when hidden.**

Toggle off per-block via the `block_type_metadata` filter:

```php
add_filter( 'block_type_metadata', function ( $metadata ) {
    if ( 'my-plugin/critical-tracker' === $metadata['name'] ) {
        $metadata['supports']['visibility'] = false;
    }
    return $metadata;
} );
```

UI: List View "Hidden" indicator + `Ctrl+Shift+H` / `⌘+Shift+H`.

### `WP_Block_Processor` — streaming block parser, read-only (6.9)

`parse_blocks()` is O(n) memory in a way that explodes — a 3 MB post measured at 14 GB peak. The streaming processor walks block delimiters in a single forward pass with O(1)-ish memory and supports early termination. Read-only in 6.9; write API planned for a later release. Existing `parse_blocks()` callers are unaffected.

```php
$processor = new WP_Block_Processor( $post_content );
while ( $processor->next_block() ) {
    if ( $processor->is_block_type( 'core/heading' ) ) {
        $attrs = $processor->allocate_and_return_parsed_attributes();
        // ...
    }
}
```

Methods: `next_block()`, `next_delimiter()`, `next_token()`, `get_block_type()`, `is_block_type()`, `opens_block()`, `get_html_content()`, `allocate_and_return_parsed_attributes()`.

### `WP_HTML_Processor::serialize_token()` is now public (6.9)

Until 6.9 you could iterate the processor but couldn't safely emit normalized HTML for the current token. Now you can build serialization passes that mutate output without round-tripping through `parse_blocks()`. Companion to `WP_Block_Processor`.

Plus new helpers `wp_js_dataset_name()` ↔ `wp_html_custom_data_attribute_name()` for round-tripping data attributes (`data-wp-bind--class` ↔ `wpBind-Class`). New test helper `assertEqualHTML()` on `WP_UnitTestCase` for semantic HTML diffing in block tests.

Security note: script tag mutations now reject content containing `<script` or `</script` to prevent script-data-state attacks.

## Script Modules and Interactivity glue

### `script_module_data_{$module_id}` filter + dynamic-import deps (6.7)

The supported way to ship server-rendered config to a `viewScriptModule`. Replaces ad-hoc `wp_localize_script()` patterns that don't work for ESM modules.

```php
add_filter( 'script_module_data_my-block/view', function ( $data ) {
    $data['apiBase']  = rest_url( 'my-plugin/v1' );
    $data['nonce']    = wp_create_nonce( 'wp_rest' );
    return $data;
} );
```

JSON-emitted into `<script type="application/json" id="wp-script-module-data-{$module_id}">`. Read at module init.

Declare deferred deps via `[ 'id' => '@wordpress/a11y', 'import' => 'dynamic' ]` in `wp_register_script_module()`. `@wordpress/scripts` with `--experimental-modules` and `@wordpress/dependency-extraction-webpack-plugin` auto-resolve these.

### Interactivity API: `getServerState()` / `getServerContext()` (6.7)

Closes a real bug: `actions.navigate()` used to silently overwrite client state with the new page's server state. Now it only adds new properties; opt in to overwrites via these subscribable read-only views.

```js
import { getServerState, getServerContext } from '@wordpress/interactivity';

// data-wp-watch callback that diffs server vs client and writes back
( { state } ) => {
    const server = getServerState();
    if ( server.cart !== state.cart ) {
        state.cart = server.cart;
    }
}
```

### Interactivity API: `data-wp-ignore` deprecated; `---suffix` unique-id syntax (6.9)

`data-wp-ignore` broke context inheritance and confused the interactivity-router on client-side navigation. The new unique-id suffix syntax lets you attach multiple directives of the same family to one element.

```html
<div
    data-wp-watch---hydrate="callbacks.hydrate"
    data-wp-watch---log="callbacks.log"
></div>
```

New TS helpers in `@wordpress/interactivity`: `AsyncAction<ReturnType>` and `TypeYield<T>` for typing yielded values in async actions without circular refs.

## Design tools rolled out per block (6.7 / 6.8)

Custom CSS targeting these blocks via class names alone breaks when users add inline border/spacing styles. Plus, theme.json `styles.blocks.<name>.border|spacing` settings now take effect on these blocks for the first time.

**6.7 border** added to: Buttons, Categories, Column, Comment Author Name, Comment Content, Comment Date, Comment Edit Link, Comment Reply Link, File, Gallery, Heading, Image, Latest Comments, List, List Item, Media & Text, Paragraph, Post Author (+ Biography, Name), Post Comments Form, Post Content, Post Date, Post Excerpt, Post Terms, Post Title, Preformatted, Query Title, Quote, Search, Site Tagline, Site Title, Social Links, Tag Cloud, Term Description.

**6.7 dimensions:** Image, Search. **6.7 background image:** Quote, Verse. **6.7 typography (`writingMode`):** Site Title, Site Tagline, Verse, button element.

**6.8 color:** Archives, Categories, Page List. **6.8 border:** Archives, Latest Posts, Page List. **6.8 dimensions:** Page List, RSS.

Source: [Roster of design tools 6.7 edition 2](https://make.wordpress.org/core/2024/10/17/roster-of-design-tools-per-block-wordpress-6-6-edition-2/), [Roster of design tools 6.8](https://make.wordpress.org/core/2025/03/12/roster-of-design-tools-per-block-wordpress-6-8-edition/)

## DataViews / DataForm — admin UI toolkit graduates (6.9)

DataViews/DataForm are now the path for building admin block-management UIs (think Site Editor's pattern manager, future block-level admin tooling). 6.9 is the inflection point where it stops being "Site Editor-only" and starts being a general toolkit.

Field types: 3 → 13 (`array`, `boolean`, `color`, `date`, `email`, `media`, `number`, `password`, `telephone`, `url` added). Edit controls: 5 → 16. Filter operators: 6 → 22. New: `readOnly` flag, async `getElements`, dot-notation `setValue`, rule-based validation (`required`, `elements`, sync/async custom).

New layouts: `groupByField`, infinite scroll, locked filters, `align`, `enableMoving`, `renderItemLink`. New `DataViewsPicker` component. New package `@wordpress/views` for view-state persistence via WP preferences. DataForm: `card` and `row` layouts, panel `openAs: 'dropdown'|'modal'`, `useFormValidity` async hook.

Source: [DataViews/DataForm in WordPress 6.9](https://make.wordpress.org/core/2025/11/11/dataviews-dataform-et-al-in-wordpress-6-9/)
