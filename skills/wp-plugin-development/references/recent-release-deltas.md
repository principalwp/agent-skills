# Recent release deltas (WP 6.7 → 6.9)

Plugin-author-relevant changes that drive code edits. Concrete deltas only — new APIs, new filters, behavior changes — sourced from Make/Core dev notes and Field Guides for 6.7, 6.8, and 6.9.

## Plugin dependencies — `Requires Plugins:` header (6.5 baseline, still current)

First-class declarative dependency contract. If your plugin extends another (WooCommerce, Yoast, ACF), stop checking `function_exists()`/`class_exists()` in `plugins_loaded` — declare it in the main file header.

```php
/**
 * Plugin Name: My WooCommerce Add-on
 * Requires Plugins: woocommerce, akismet
 */
```

Core blocks activation until dependencies are active, blocks deletion of dependencies while dependents exist, and surfaces install/activate UI. Slugs cannot contain commas. **No version constraints** — fall back to runtime version checks if you need a specific version. WordPress.org-hosted plugins may only depend on other WordPress.org-hosted slugs; non-hosted plugins can depend on either. Inspect status via the `WP_Plugin_Dependencies` class.

Source: [Plugin Dependencies in 6.5](https://make.wordpress.org/core/2024/03/05/introducing-plugin-dependencies-in-wordpress-6-5/)

## `register_block_template()` — plugins ship block templates without a child theme (6.7)

Plugins shipping CPTs (event archive, product layouts) previously had to fight `template_include` to provide a default block-theme template. Now you can register one declaratively.

```php
register_block_template( 'my-plugin//event-archive', [
    'title'       => __( 'Event Archive', 'my-plugin' ),
    'description' => __( 'Default archive layout for events.', 'my-plugin' ),
    'content'     => '<!-- wp:query {...} --> ... <!-- /wp:query -->',
    'post_types'  => [ 'event' ],
] );
```

Theme templates always win; this is a fallback layer. Block template *parts* are NOT registerable yet (explicit limitation). Pair with `unregister_block_template()` on deactivation.

Source: [Plugin template registration API in 6.7](https://make.wordpress.org/core/2024/10/20/new-plugin-template-registration-api-in-wordpress-6-7/)

## Speculation Rules ship by default — opt out stateful URLs (6.8)

Core ships Speculation Rules JSON on every public frontend page by default. **Plugins with stateful URLs (cart, checkout, logout, custom AJAX-ish endpoints) must opt those URLs out** or risk side-effect-on-prefetch bugs.

Default config: mode `prefetch`, eagerness `conservative` (mousedown/touchstart). Disabled when user is logged in OR pretty permalinks are off.

```php
// Disable everywhere for the request
add_filter( 'wp_speculation_rules_configuration', '__return_null' );

// Or just exclude paths
add_filter( 'wp_speculation_rules_href_exclude_paths', function ( $paths, $mode ) {
    return array_merge( $paths, [
        '/cart/*',
        '/checkout/*',
        '/my-account/*',
        '/?logout=*',
    ] );
}, 10, 2 );

// Or contribute additional rules
add_action( 'wp_load_speculation_rules', function ( $rules ) {
    $rules->add_rule( 'prefetch', 'my-priority', [
        'source'    => 'list',
        'urls'      => [ '/key-landing-page/' ],
        'eagerness' => 'eager',
    ] );
} );
```

UI escape hatch: editors can add CSS class `no-prefetch` or `no-prerender` to any block.

Source: [Speculative Loading in 6.8](https://make.wordpress.org/core/2025/03/06/speculative-loading-in-6-8/)

## bcrypt is the default password hash (6.8)

Plugins that store user passwords (membership, custom auth, migration) or duplicate WP's hashing must stop hardcoding phpass.

`wp_hash_password()` now produces bcrypt (`$2y$...`). `wp_check_password()` transparently verifies legacy phpass (`$P$...`) hashes and rehashes on successful match — same for the Application Password verification path. **If your plugin compares stored hashes directly, switch to `wp_check_password()`.**

Source: [WordPress 6.8 will use bcrypt for password hashing](https://make.wordpress.org/core/2025/02/17/wordpress-6-8-will-use-bcrypt-for-password-hashing/)

## Translation loading: drop `load_plugin_textdomain()` (6.7+)

Calling translation functions before `init` now triggers `_doing_it_wrong()`. Plugins with `__()` in constructors / class-load time emit notices.

- WP automatically handles textdomain loading for plugins on WordPress.org (or those declaring `Text Domain` and `Domain Path` headers + `Requires at least: 6.7`).
- `load_plugin_textdomain()` and `load_theme_textdomain()` defer to JIT loading — explicit calls become no-ops.
- New `has_translation( $singular, $domain = 'default', $context = '' )` — checks existence without forcing load.
- Admin emails to `admin_email` use the matching user's locale (not site locale) when a user record exists.

Plugins targeting `Requires at least: 6.8`+ can drop `load_plugin_textdomain()` entirely.

`.l10n.php` translation files (6.5+ baseline) are now standard — opcache-backed, faster to load than `.mo`. Generate via `wp i18n make-php`. `.mo` remains supported as fallback.

Source: [i18n improvements 6.7](https://make.wordpress.org/core/2024/10/21/i18n-improvements-6-7/), [i18n improvements 6.8](https://make.wordpress.org/core/2025/03/12/i18n-improvements-6-8/)

## HTML API matured — content rewriting plugins should switch (6.7 / 6.9)

Plugins doing content rewriting (lazy-load, link rewriting, srcset injection, inline schema) can finally edit text nodes safely and parse complete documents.

**6.7:**
- `WP_HTML_Processor::create_full_parser( $html )` parses `<!DOCTYPE>` through `</html>`. Previously only `create_fragment()` existed.
- `set_modifiable_text( $new )` replaces text content of the current node — context-aware escaping for SCRIPT/STYLE/TITLE/comments.
- `WP_HTML_Processor::normalize( $html )` returns a well-formed equivalent.
- `set_attribute()` returns `false` when WP rejects the update (was silent).
- Tag Processor scan 3.5–7.5% faster.

**6.9:**
- `WP_HTML_Processor::serialize_token()` is now public. Build serialization passes that mutate output without round-tripping `parse_blocks()`.
- `wp_js_dataset_name( $html_attr )` and `wp_html_custom_data_attribute_name( $js_name )` — round-trip data attributes.
- Constructors safely cast `null` to `''`. Static `create_*` factories return `null` and emit `_doing_it_wrong()` on invalid input.
- `set_modifiable_text()` on `<script>` rejects payloads containing `<script` / `</script>` (script-data-double-escaped-state attacks).
- New test helper `WP_UnitTestCase::assertEqualHTML()` for semantic HTML diffs.

Source: [HTML API in 6.7](https://make.wordpress.org/core/2024/10/17/updates-to-the-html-api-in-6-7/), [HTML API in 6.9](https://make.wordpress.org/core/2025/11/21/updates-to-the-html-api-in-6-9/)

## `WP_Block_Processor` — streaming block parser (6.9)

Replacement for the regex-based `parse_blocks()` for plugins that walk/modify post content blocks (analytics, rewriters, migration). `parse_blocks()` is O(n) memory in a way that explodes — a 3 MB post measured at 14 GB peak. Streaming = O(1)-ish memory, structural correctness on malformed input, parsed JSON attributes. Read-only in 6.9; coexists with `parse_blocks()` (not deprecated).

```php
$processor = new WP_Block_Processor( $post->post_content );
while ( $processor->next_block() ) {
    if ( $processor->is_block_type( 'core/heading' ) ) {
        $attrs = $processor->allocate_and_return_parsed_attributes();
        // ...
    }
}
```

Source: [Streaming block parser in 6.9](https://make.wordpress.org/core/2025/11/19/introducing-the-streaming-block-parser-in-wordpress-6-9/)

## `WP_Query` cache key generation changed (6.9)

Breaking for plugins that build their own object-cache keys mirroring core's query cache, or that warm caches by precomputing keys. Persistent object cache drop-ins are unaffected.

Core changed the canonicalization used to derive cache keys for `WP_Query`-driven queries so semantically identical queries hash identically. **Use the four new helper functions in core to generate keys instead of rolling your own** — they ensure parity.

Source: [Consistent cache keys for query groups in 6.9](https://make.wordpress.org/core/2025/11/17/consistent-cache-keys-for-query-groups-in-wordpress-6-9/)

## `register_post_type()` gains `embeddable`; `wp_next_scheduled` becomes a filter (6.8)

```php
register_post_type( 'paid_post', [
    'public'      => true,
    'embeddable'  => false,  // exclude from oEmbed even though public
    /* ... */
] );

// Virtualize cron timestamps without touching option storage
add_filter( 'wp_next_scheduled', function ( $ts, $hook, $args ) {
    if ( 'my_dynamic_cron' === $hook ) {
        return time() + my_dynamic_offset();
    }
    return $ts;
}, 10, 3 );
```

`embeddable` defaults to the value of `public` (so existing CPTs get the new behavior implicitly — audit any CPT where you want a dark-launch through oEmbed).

Other 6.8 misc that bit real plugins:
- `import_filters` action on `/wp-admin/import.php` for adding importer entries
- `wp_editor_set_quality` filter receives a `$size` arg (`['width' => ..., 'height' => ...]`) for per-thumbnail quality
- `wp_prevent_unsupported_mime_type_uploads` filter — return `false` to allow exotic uploads
- `image_max_bit_depth` filter — Imagick HDR depth control
- `setted_transient` action renamed to `set_transient`; old name still fires but is deprecated
- `wp_video_shortcode()` / `wp_audio_shortcode()` emit boolean attributes (`loop`, `autoplay`, `muted`) without `="value"` for HTML5 compliance
- Body classes added: `wp-theme-{slug}`, `wp-child-theme-{slug}`, `wp-singular`

Source: [Misc developer changes in 6.8](https://make.wordpress.org/core/2025/03/25/miscellaneous-developer-changes-in-wordpress-6-8/)

## Abilities API — register plugin actions, not REST routes (6.9)

The standard registry for "things this site can do." Consumed by AI agents (MCP), CLI, and the future client-side editor. **Plugins should expose major actions (create product, moderate comment, sync data) as abilities rather than ad-hoc REST endpoints.**

```php
add_action( 'wp_abilities_api_init', function () {
    wp_register_ability( 'my-plugin/sync-orders', [
        'label'              => __( 'Sync orders', 'my-plugin' ),
        'category'           => 'commerce',
        'execute_callback'   => fn ( $input ) => MyPlugin\sync_orders( $input ),
        'permission_callback'=> fn ( $input ) => current_user_can( 'manage_woocommerce' ),
        'input_schema'       => [ /* JSON Schema */ ],
        'output_schema'      => [ /* JSON Schema */ ],
        'meta'               => [
            'show_in_rest' => true,
            'annotations'  => [
                'readonly'    => false,
                'destructive' => false,
                'idempotent'  => true,
            ],
        ],
    ] );
} );
```

When `show_in_rest => true`, the ability is invokable at `/wp-json/wp-abilities/v1/...` and the `permission_callback` is enforced. Schema-driven validation runs automatically. WP-CLI: `wp ability ...`.

See the `wp-abilities-api` skill for full coverage.

## URL escaping default protocol can be HTTPS (6.9)

Plugins that pass `$protocols` to `esc_url()` for whitelist purposes can now flip the protocol-less default from `http://` to `https://`.

```php
esc_url( $maybe_url, [ 'https', 'http' ] );  // prepends https:// when no scheme present
```

`esc_url_raw()` and `sanitize_url()` follow the same rule. No change when `$protocols` is omitted (back-compat preserved).

## `wp_mail()` inline images via `cid:` + sender header isolation (6.9)

Transactional plugins (notifications, membership, WooCommerce) can stop hosting inline assets on a CDN.

```php
wp_mail(
    'recipient@example.com',
    'Subject',
    '<p>Hi! See attached: <img src="cid:logo">.</p>',
    [ 'Content-Type: text/html' ],
    [ '/path/to/logo.png' ]   // attached + Content-ID auto-wired
);
```

Plus: PHPMailer state (custom headers, attachments) is now reset between calls — a header set in one `wp_mail()` no longer leaks into the next. Sender (`From:`) is set extensibly per-call.

## Admin menu search source: `$_SERVER['QUERY_STRING']` → `$_GET` (6.9)

Breaking for plugins that hooked into admin menu rendering and inspected `QUERY_STRING` for the search filter (multi-network admin extenders). Switch to `$_GET`.

## Frontend perf: `fetchpriority` + footer args for scripts and modules (6.9)

Plugins shipping non-critical UI scripts (chat widgets, analytics, social embeds) should mark them `fetchpriority => 'low'` to claw back LCP.

```php
wp_register_script( 'my-chat-widget', ..., [
    'fetchpriority' => 'low',
    'in_footer'     => true,
] );

// Modules now support footer printing too
wp_register_script_module( 'my-plugin/view', ..., [
    'fetchpriority' => 'low',
    'in_footer'     => true,
] );

// Mutate later
wp_script_add_data( $handle, 'fetchpriority', 'low' );
WP_Script_Modules::set_fetchpriority( $id, 'low' );
WP_Script_Modules::set_in_footer( $id, true );
```

Core auto-applies `fetchpriority="low"` to Interactivity API view modules and the `comment-reply` script. Emoji detection moved from blocking inline `<script>` to deferred footer module (~3KB freed). **`_wpemojiSettings` global may not exist before `DOMContentLoaded`** — plugins reading it must wait.

Source: [WordPress 6.9 frontend performance field guide](https://make.wordpress.org/core/2025/11/18/wordpress-6-9-frontend-performance-field-guide/)

## Template enhancement output buffer — clean replacement for `ob_start()` hacks (6.9)

Core finally gives plugins a hook to mutate the entire rendered HTML buffer with `WP_HTML_Tag_Processor` — replacing decade-old hacks built on `ob_start()` in `template_redirect`.

```php
add_filter( 'wp_template_enhancement_output_buffer', function ( $html ) {
    $p = new WP_HTML_Tag_Processor( $html );
    while ( $p->next_tag( 'img' ) ) {
        $p->set_attribute( 'loading', 'lazy' );
    }
    return $p->get_updated_html();
} );
```

Companions:

- `wp_template_enhancement_output_buffer_started` action — buffer just opened (set headers)
- `wp_finalized_template_enhancement_output_buffer` action — last call before flush; can still send HTTP headers
- `wp_should_output_buffer_template_for_enhancement` filter — control activation (force on with `__return_true`)
- `wp_before_include_template` action

Plus, classic themes can opt into per-block CSS via `should_load_separate_core_block_assets` (was block-themes-only). `styles_inline_size_limit` default raised to 40KB (was 20KB). `enqueue_empty_block_content_assets( bool, $block_name )` filter forces assets to load for hidden/empty blocks. `oembed_discovery_links` filter to disable auto-printed oEmbed `<link>` tags. **WP-Cron execution moved from `init` to `shutdown`** — TTFB win.

## Block API version 3 required (6.9 deprecation, 7.0 enforcement)

Plugins still on `apiVersion: 1` or `2` log a console warning in 6.9 and break in 7.0 when the post editor canvas becomes a full iframe. `block.json` schema now only accepts `apiVersion: 3` for new/updated blocks.

Migration: ensure your block plays well inside the iframe (no parent-document DOM access; `useRefEffect` pattern; load styles via block.json so they reach the iframe).

## IE conditional comments removed from script/style API (6.9)

Plugins still calling `wp_style_add_data( $handle, 'conditional', 'lt IE 9' )` (or similar) silently drop the conditional and emit the asset for all browsers. Support for the `conditional` data key on `wp_register_script` / `wp_register_style` is removed.

## UTF-8 modernization — pure-PHP fallback (6.9)

Plugins handling user-submitted text (forms, imports, social feeds) get consistent UTF-8 normalization regardless of `mbstring`/`iconv` availability. **Behavior is now stable across hosts; do not assume mbstring is required to support emoji/accents.** Removes a class of "works on host A, breaks on host B" bugs.
