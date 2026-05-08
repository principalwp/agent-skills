# Block theme release deltas (WP 6.7 → 6.9)

theme.json keys, block hooks, pattern semantics, and style variations changes a block theme must reckon with. theme.json `version` is still `3` (introduced in 6.6); 6.7/6.8/6.9 ship under v3 with additive keys.

## theme.json v3 default-presets (anchor — bites every v2 → v3 migration)

Breaking behavior change you may meet on inherited themes. In v2, theme `fontSizes` / `spacingSizes` slugs replaced core presets. In v3, they **coexist with core defaults** unless explicitly opted out:

```json
{
    "version": 3,
    "settings": {
        "typography": { "defaultFontSizes": false },
        "spacing":    { "defaultSpacingSizes": false }
    }
}
```

Default v3 size slugs are `small`/`medium`/`large`/`x-large` — `normal` and `huge` are gone. Affects emitted `--wp--preset--font-size--*` custom properties and the size picker.

## theme.json `settings.background` + global `styles.background` (6.6 anchor, expanded in 6.7)

Background image is now a first-class top-level style property — settable site-wide via `styles.background` and per-block via `styles.blocks.<name>.background`. Pre-6.6 it was a Cover/Group-only attribute.

```json
"styles": {
    "background": {
        "backgroundImage":      { "url": "file:./assets/hero.jpg" },
        "backgroundSize":       "cover",
        "backgroundRepeat":     "no-repeat",
        "backgroundAttachment": "fixed",
        "backgroundPosition":   "center"
    }
}
```

6.7 expanded `supports.background.backgroundImage` to Quote, Pull Quote, Verse, Post Content. UI exposes the global background image picker in Styles → Background.

## theme.json `settings.shadow.presets` site-wide (matured 6.6 → 6.8)

```json
"settings": {
    "shadow": {
        "defaultPresets": false,
        "presets": [
            { "name": "Sharp", "slug": "sharp", "shadow": "0 4px 0 0 #000" }
        ]
    }
}
```

Emits `--wp--preset--shadow--<slug>` for use in patterns and custom CSS. Section styles can override per-block-style-variation.

## theme.json `settings.border.radiusSizes` presets (6.9)

Border-radius joins font-size and spacing as a preset-driven design token.

```json
"settings": {
    "border": {
        "radiusSizes": [
            { "name": "Soft", "slug": "soft", "size": "4px" },
            { "name": "Pill", "slug": "pill", "size": "9999px" }
        ]
    }
}
```

UI: 1–8 presets render as a slider with stops; ≥9 presets render as a dropdown; custom input is always available. Emits `--wp--preset--border-radius--<slug>`. Purely additive — themes without `radiusSizes` keep existing behavior.

## theme.json forms support — `styles.elements.textInput` and `select` (6.9)

First time `<input>`, `<textarea>`, and `<select>` are addressable from theme.json. Cascades across third-party form plugins (Gravity Forms, WPForms, Contact Form 7) wherever they emit standard tags.

```json
"styles": {
    "elements": {
        "textInput": {
            "border":     { "radius": "var:preset|border-radius|soft", "width": "1px" },
            "color":      { "background": "var:preset|color|surface", "text": "var:preset|color|on-surface" },
            "spacing":    { "padding": { "top": "0.75rem", "right": "1rem", "bottom": "0.75rem", "left": "1rem" } },
            "typography": { "fontSize": "var:preset|font-size|medium" }
        },
        "select": { /* same shape */ }
    }
}
```

`textInput` covers both `input[type=text|email|url|password|number]` and `textarea`. **Limitation:** no pseudo-state styling — `:focus`, `:hover`, `:disabled` still require custom CSS via `styles.css` or block CSS.

## Button typography inheritance from Global Styles (6.9)

Pre-6.9, `.wp-element-button` (shared by core blocks and many plugins) ignored `styles.typography` overrides set in Global Styles. Themes had to duplicate typography inside `styles.elements.button`.

In 6.9, when a user adjusts typography in Global Styles, `.wp-element-button` inherits. Theme-level `styles.elements.button.typography` still wins where set. **If your theme intentionally diverged button typography from base typography, audit `styles.elements.button` — Global Styles changes can now bleed in.**

## Block style variations as JSON partials in `/styles/blocks/`

Three registration paths now coexist:

1. `register_block_style()` (PHP)
2. `theme.json` `styles.blocks.<name>.variations`
3. JSON partial in `/styles/blocks/<name>/<slug>.json`

The partial path is the only one that can target multiple block types via top-level `blockTypes`:

```json
{
    "$schema": "https://schemas.wp.org/wp/6.7/theme.json",
    "version": 3,
    "title":   "Outlined",
    "slug":    "outlined",
    "blockTypes": [ "core/group", "core/columns" ],
    "styles": { "border": { "width": "2px", "color": "currentColor" } }
}
```

Known regression to watch (Gutenberg #72854 / #73527): JSON-partial variations may emit no CSS for some inner-block selectors (e.g. List Item). Verify in 6.8/6.9 before shipping.

## Section Styles — block style variations applied to nested blocks/elements

"Section styles" is **not a separate API** — it is the block-style-variation system, with the new semantic that nested blocks and elements inherit the variation's styles. Theme authors looking for a new key won't find one.

```json
{
    "title": "Dark Hero",
    "slug": "dark-hero",
    "blockTypes": [ "core/group" ],
    "styles": {
        "color": { "background": "#000", "text": "#fff" },
        "blocks": {
            "core/heading": { "color": { "text": "#ffd700" } }
        },
        "elements": {
            "link": { "color": { "text": "#90caf9" } }
        }
    }
}
```

6.8 added a section-style switcher in the zoom-out toolbar — users can swap variations inline without entering the section. This is the multi-brand unlock for enterprises: a single Group block style variation cascades to nested headings, paragraphs, images. Combined with multiple style variations, this is the path to "one theme, N brands."

## Block Hooks — Template Part `firstChild`/`lastChild` (6.7) + post content + synced patterns (6.8)

**6.7:** `"core/template-part": "firstChild"` / `"lastChild"` in `blockHooks` finally works on template parts (was silently ignored — only `before`/`after` worked). Template Part joins `core/navigation` as the second "special" block whose contents are loaded from a separate post and whose `ignoredHookedBlocks` list is stored in `_wp_ignored_hooked_blocks` post meta (not in serialized markup).

**6.8:** Block Hooks now apply to **regular post content** and **synced patterns** (`wp_block` post type). Plugins relying on the "post content is untouched" assumption see surprise blocks. Same per-context dedup rule applies: synced patterns count as their own context, so `multiple: false` does not deduplicate across the surrounding template.

`hooked_block_types` filter signature: `($hooked_block_types, $relative_position, $anchor_block_type, $context)` where `$context` is the template / template part / `wp_navigation` post / pattern.

## Block Hooks honor `supports.multiple: false` (6.7)

Hooked blocks declaring `"multiple": false` insert only if not already present in the current context. **Per-context, not per-page** — a single rendered page can still contain multiple instances if multiple contexts (template + template part + navigation post) each contribute one.

## `ignoredHookedBlocks` — two storage paths

You can't pick. For inline anchor blocks, `ignoredHookedBlocks` lives in the anchor block's `metadata` attribute. For blocks whose content is loaded externally (template parts, navigation, synced patterns), it's persisted in `_wp_ignored_hooked_blocks` post meta on the host post.

Diffing/migrating template parts requires knowing both. Treat the meta key as part of the part's identity for round-trip serialization.

## Starter patterns: per-CPT + recursive `/patterns/` subfolders (6.8)

Pattern file header gains `Post Types: post, page, my_cpt` (CSV). Patterns assigned `Block Types: core/post-content` show in the Starter Content modal for every listed post type — was Pages-only.

The `/patterns/` directory walker is now recursive — `/patterns/header/site-header.php` registers identically to `/patterns/site-header.php`. Organize as you like.

## Block Bindings — pattern overrides expanded to image caption (6.9)

Pattern overrides (synced-pattern-with-editable-fields) previously covered Heading text, Paragraph content, Image url, and Image alt. 6.9 adds `caption` for `core/image`. No schema change — the override system was simply taught to recognize the caption attribute.

```json
"metadata": { "bindings": { "caption": { "source": "core/pattern-overrides" } } }
```

## Block Bindings — `block_bindings_supported_attributes_{$block_type}` filter (6.9)

Lets a theme widen or narrow which attributes of a given block are eligible for bindings. Use case: enabling `core/cover` `url` binding to a meta field, or restricting `core/image` `alt` from being bound on a locked theme.

```php
add_filter( 'block_bindings_supported_attributes_core/cover', function ( $attrs ) {
    $attrs[] = 'url';                  // expose url for binding
    return $attrs;
} );
```

Returned array overrides core defaults entirely (replace, not merge).

## Multi-brand / governance patterns worth knowing

These came up consistently in agency-grade write-ups during the window. Not deltas — production techniques.

### VIP Block Governance — role-aware editor governance via `governance-rules.json`

The `Automattic/vip-governance-plugin` ships a `governance-rules.json` cascade that mirrors `theme.json`. Rule types: `default`, `role`, `postType`. Lets you lock down `allowedBlocks`, `blockSettings`, and design-token availability per role and per post type without writing PHP. The closest thing to a built-in role-aware editor governance schema. Not core, but adopted at scale by Gold-partner agencies.

### Block locking caveat — any user with `edit-blocks` can unlock

By default, `templateLock: 'all'` and per-block `lock: { move: true, remove: true }` can be unlocked by any user with `edit-blocks`. To deny non-admins the unlock UI:

```php
add_filter( 'block_editor_settings_all', function ( $settings, $context ) {
    if ( ! current_user_can( 'manage_options' ) ) {
        $settings['canLockBlocks'] = false;
    }
    return $settings;
}, 10, 2 );
```

### `templateLock: 'contentOnly'` requires attribute `role: 'content'`

For `contentOnly` to expose any editable surface, the block must have at least one attribute marked `role: 'content'`. Custom blocks without content-role attributes silently break in Write/Design modes — an issue WP 7.0 amplifies by making `contentOnly` the default for unsynced patterns.

### Disable the Font Library UI on client builds

Default Font Library UI lets editors install fonts from Google Fonts at runtime — undesirable on regulated/enterprise sites. Disable:

```php
add_filter( 'block_editor_settings_all', function ( $settings ) {
    $settings['fontLibraryEnabled'] = false;
    return $settings;
} );
```

10up's `Engineering-Best-Practices` includes this as a default for client themes.

## Sources

- [theme.json version 3 (anchor)](https://make.wordpress.org/core/2024/06/19/theme-json-version-3/)
- [Block Hooks in 6.7](https://make.wordpress.org/core/2024/10/21/updates-to-block-hooks-in-wordpress-6-7/)
- [Block Bindings 6.9](https://make.wordpress.org/core/2025/11/12/block-bindings-improvements-in-wordpress-6-9/)
- [theme.json border radius presets in 6.9](https://make.wordpress.org/core/2025/11/12/theme-json-border-radius-presets-support-in-wordpress-6-9/)
- [theme.json forms in 6.9](https://developer.wordpress.org/news/2025/11/how-wordpress-6-9-gives-forms-a-theme-json-makeover/)
- [Section styles (6.6)](https://developer.wordpress.org/news/2024/06/styling-sections-nested-elements-and-more-with-block-style-variations-in-wordpress-6-6/)
- [WordPress 6.9 Field Guide](https://make.wordpress.org/core/2025/11/25/wordpress-6-9-field-guide/)
- [Big Bite — Enterprise brand governance at scale](https://bigbite.net/2025/05/08/wordpress-full-site-editing-enterprise-brand-governance-at-scale/)
- [VIP Governance plugin](https://github.com/Automattic/vip-governance-plugin)
- [10up Engineering Best Practices](https://10up.github.io/Engineering-Best-Practices/)
