# Block API internals and gotchas

## GLOBAL_ATTRIBUTES

Two attributes exist on every block: `lock` (object) and `metadata` (object). These are **not** `className` or `anchor` (those come from `supports`).

## render.php variables vs render_callback signature

The `render.php` template receives exactly three variables:

| Variable | Type | Description |
|----------|------|-------------|
| `$attributes` | `array` | Block attributes |
| `$content` | `string` | Rendered inner blocks / content |
| `$block` | `WP_Block` | The WP_Block instance |

`render_callback` receives 3 arguments in a different shape: `$attributes`, `$block_content`, `$this` (WP_Block). Code that copy-pastes between the two without renaming `$content` ↔ `$block_content` silently breaks.

## Block type metadata filters

Two filters, applied in order:
1. `block_type_metadata` — raw block.json data (before processing)
2. `block_type_metadata_settings` — processed settings (after parsing)

Use the first to rewrite block.json before WordPress processes it; use the second to override the resulting registration.

## is_dynamic() check

`WP_Block_Type::is_dynamic()` checks `is_callable($this->render_callback)` — not the existence of a `render` field or any `supports` flag. If you registered a `render` path but the file produced no callable (e.g., file missing, syntax error), `is_dynamic()` quietly returns `false` and the block falls back to its (probably empty) `save()` output.

## skip_inner_blocks

When `skip_inner_blocks = true`, inner blocks are not pre-rendered before the render callback runs — the callback is responsible for rendering them. Used by `core/post-template`.

## Deprecation entry keys

Exactly 6 valid keys in a deprecation entry: `attributes`, `supports`, `save`, `migrate`, `isEligible`, `apiVersion`. Other keys are silently ignored.

## Block hooks

Four valid positions: `before`, `after`, `first_child`, `last_child`. In `block.json`, use camelCase: `blockHooks`.

**Scope**: Block hooks only apply to templates, template parts, patterns, and navigation posts — **NOT to post content**. A hook targeting `core/post-content` won't appear inside individual posts.

**Priority on `the_content`**: Block hooks at priority 8, `do_blocks` at priority 9. A `the_content` filter at priority < 8 runs against pre-block markup; > 9 runs against rendered HTML.

**`ignoredHookedBlocks`**: When a user removes a hooked block, the block name is added to `ignoredHookedBlocks` on the anchor's metadata, preventing auto-reinsertion. For templates: stored inline in `attrs.metadata`. For posts/patterns: stored in post meta `_wp_ignored_hooked_blocks`.

**`multiple: false`**: If an instance of the hooked block already exists in context, the hook is skipped.

**Not backfilled**: Hooks are applied at activation time, not retroactively to existing templates. New hook registrations only affect content created or edited afterwards.

## Block bindings (6.9)

Four built-in sources: `post-meta`, `pattern-overrides`, `post-data`, `term-data` (last two new in 6.9).

Bindings stored at `attrs.metadata.bindings` (metadata is a GLOBAL_ATTRIBUTE).

`get_value_callback` signature: `function($source_args, $block_instance, $attribute_name)`.

Filter after callback: `block_bindings_source_value` (since 6.7, 5 arguments).

For `core/post-meta` to work: `show_in_rest` must be `true`, meta must not be protected, and post must be publicly viewable or user must have `read_post`.

`__default` key in pattern-overrides expands to ALL supported attributes of the block type.

Blocks supporting bindings by default: paragraph, heading, image, button, post-date, navigation-link, navigation-submenu.

## Block visibility (6.9)

`supports.visibility` enables a visibility toggle. When `blockVisibility = false`, the block content is suppressed at render time.

## Block categories

Default categories: text, media, design, widgets, theme, embed, reusable — **`navigation` is NOT a default category**. Adding a navigation-related block to category `'navigation'` makes it disappear from the inserter unless the category is registered first.

Filter: `block_categories_all` (since 5.8, replaces deprecated `block_categories`).

## HTML API quick reference

### WP_HTML_Tag_Processor (6.2+)

Flat tag-by-tag iteration. Use for simple attribute manipulation and tag removal.

```php
$tp = WP_HTML_Tag_Processor::create_fragment( $html );
while ( $tp->next_tag( 'script' ) ) {
    $tp->remove_tag();
}
return $tp->get_updated_html();
```

- `get_attribute('disabled')` returns PHP `true` for boolean attrs, `null` if missing, string for valued attrs.
- `get_tag()` returns **uppercase** (`'DIV'`, `'SPAN'`) regardless of source casing — string-compare against uppercase only.

### WP_HTML_Processor (6.4+)

Full tree construction. Extends Tag_Processor. Use for nested element handling.

- `create_fragment()` context element is `<body>` only — passing other context returns `null`.
- `normalize($html)` = `create_fragment($html)->serialize()` — round-trips through the HTML5 parser to canonicalize markup before comparison.

### 6.9 additions

- `serialize_token()` — now public; returns normalized, well-formed serialization of the current token.
- `set_modifiable_text($text)` — replaces text content of modifiable elements (e.g., `<script>`). Returns `false` if the replacement contains nested script markup (security hardening).
- `remove_tag()` / `remove_token()` — strips elements from output.

### Preferred over regex/DOMDocument

WordPress officially considers WP_HTML_Tag_Processor / WP_HTML_Processor the correct approach for HTML manipulation. Use them in place of `preg_match`, `preg_replace`, and `DOMDocument` — those are explicitly disallowed by current WordPress core HTML-processing tests.

### Dataset helper functions

- `wp_html_custom_data_attribute_name($prop)` — converts camelCase dataset property to `data-*` attribute name.
- `wp_js_dataset_name($attr)` — converts `data-*` attribute back to camelCase property.
- `get_url_in_content($html)` — extracts first URL from HTML content (preferred over regex).

## Custom block binding sources

Registered in PHP via `register_block_bindings_source()` at `init`:

```php
add_action( 'init', function() {
    register_block_bindings_source( 'my-plugin/current-time', [
        'label'              => __( 'Current time' ),
        'get_value_callback' => function( $source_args, $block_instance, $attribute_name ) {
            $format = $source_args['format'] ?? 'c';
            return wp_date( $format );
        },
        'uses_context'       => [ 'postId', 'postType' ],
    ] );
} );
```

- `uses_context` exposes block context (e.g., `postId` inside a query loop) via `$block_instance->context`.
- Return `null` to leave the original attribute value untouched; return a string to override.
- Custom sources do NOT require `show_in_rest` meta the way `core/post-meta` does — they bypass REST entirely for render.
- For the editor to preview bound values, mirror-register JS-side via `registerBlockBindingsSource` with `getValues` (and `setValues` if editable). Skip JS if bindings are frontend-read-only.
- Plan editor/frontend parity: without JS hydration, the editor shows the raw attribute; the frontend shows the callback result. The drift confuses editors.

## Static → dynamic block conversion

Path for converting a static block (saved markup) to dynamic (server-rendered) without breaking existing posts:

1. Keep all current attributes unchanged — don't rename or drop any.
2. Add `render` to `block.json` pointing at `render.php` (or `render_callback` at registration).
3. Change `save()` to `return null` (or remove the save module).
4. Add a `deprecated` entry preserving the old `save` + `attributes`. Without it, every existing post shows "Invalid block" on load.
5. In `render.php`, emit markup matching the old `save` output as closely as possible — drift reflows frontend CSS.
6. Audit classes: static blocks bake class names into saved HTML; dynamic blocks must apply them via `get_block_wrapper_attributes()`. Classes CSS relies on must be replicated server-side.

The deprecation's `save` doesn't need a `migrate` — attributes haven't changed. Its sole job is to parse the old saved HTML as valid so the editor doesn't flag it.
