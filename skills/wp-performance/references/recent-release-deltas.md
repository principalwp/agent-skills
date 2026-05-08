# Performance release deltas (WP 6.7 → 6.9)

Performance-relevant changes a backend engineer should act on. Anchored to Make/Core dev notes and the 6.9 Frontend Performance Field Guide.

## Speculation Rules ship by default in core (6.8)

Core ships Speculation Rules JSON on every public frontend page. Default config: mode `prefetch`, eagerness `conservative` (mousedown/touchstart). Disabled when user is logged in OR pretty permalinks are off.

**Performance impact:** real LCP wins on link-heavy navigation; **real correctness risk** for sites with stateful URLs because prefetch issues a real GET. Cart, checkout, logout, custom AJAX-style endpoints — opt them out.

```php
// Disable for the request
add_filter( 'wp_speculation_rules_configuration', '__return_null' );

// Or exclude paths
add_filter( 'wp_speculation_rules_href_exclude_paths', function ( $paths, $mode ) {
    return array_merge( $paths, [ '/cart/*', '/checkout/*', '/?logout=*' ] );
}, 10, 2 );

// Or contribute additional rules
add_action( 'wp_load_speculation_rules', function ( $rules ) {
    $rules->add_rule( 'prerender', 'hot', [
        'source' => 'list', 'urls' => [ '/landing/' ], 'eagerness' => 'eager',
    ] );
} );
```

UI escape hatch: editors add CSS class `no-prefetch` or `no-prerender` to any block.

Source: [Speculative Loading in 6.8](https://make.wordpress.org/core/2025/03/06/speculative-loading-in-6-8/)

## Salted cache keys via pluggable (6.9)

`wp_cache_get_salted()`, `wp_cache_set_salted()`, etc. let you scope cache keys to a salt that varies independently of the cache group — useful when you need to invalidate a slice of cached data without flushing a whole group.

```php
// Scope theme-specific cache to current stylesheet
$salt = get_stylesheet();
$value = wp_cache_get_salted( 'menu_html', 'theme_data', $salt );
if ( false === $value ) {
    $value = build_menu_html();
    wp_cache_set_salted( 'menu_html', $value, 'theme_data', $salt );
}
```

When the salt changes (e.g. theme switch), all entries under the old salt naturally fall out of LRU without needing a flush.

## `WP_Query` cache key generation changed — semantically identical queries now hash identically (6.9)

Breaking for plugins that build their own object-cache keys mirroring core's, or that warm caches by precomputing keys. Persistent object cache drop-ins are unaffected.

Core canonicalized the key derivation so semantically identical queries hash identically. Use the four new helper functions in core to generate keys instead of rolling your own — they ensure parity. **Audit any code that touches `last_changed` cache keys directly.**

Source: [Consistent cache keys for query groups in 6.9](https://make.wordpress.org/core/2025/11/17/consistent-cache-keys-for-query-groups-in-wordpress-6-9/)

## `WP_Block_Processor` — streaming block parser (6.9)

`parse_blocks()` is O(n) memory in a way that explodes — a 3 MB post measured at 14 GB peak. The streaming processor walks block delimiters in a single forward pass with O(1)-ish memory and supports early termination. **Use this for any plugin that walks post content in bulk (analytics, rewriters, migration tools).** Read-only in 6.9; coexists with `parse_blocks()`.

```php
$processor = new WP_Block_Processor( $post->post_content );
while ( $processor->next_block() ) {
    if ( $processor->is_block_type( 'core/heading' ) ) {
        // O(1) memory per iteration
    }
}
```

## `fetchpriority` + `in_footer` on script modules (6.9)

Script modules now accept the same perf args as classic scripts. Core auto-applies `fetchpriority="low"` to Interactivity API view modules and the `comment-reply` script.

```php
wp_register_script( 'my-chat-widget', $src, $deps, $ver, [
    'fetchpriority' => 'low',
    'in_footer'     => true,
] );

wp_register_script_module( 'my-block/view', $src, $deps, $ver, [
    'fetchpriority' => 'low',
    'in_footer'     => true,
] );
```

Mutate later via `wp_script_add_data( $handle, 'fetchpriority', 'low' )` or the `WP_Script_Modules::set_fetchpriority()` / `set_in_footer()` setters.

Emoji detection moved from blocking inline `<script>` to deferred footer module (~3KB freed). **`_wpemojiSettings` global may not exist before `DOMContentLoaded`** — code reading it must wait.

Source: [WordPress 6.9 Frontend Performance Field Guide](https://make.wordpress.org/core/2025/11/18/wordpress-6-9-frontend-performance-field-guide/)

## Template enhancement output buffer — replace `ob_start()` hacks (6.9)

Core finally exposes a hook to mutate the entire rendered HTML buffer with `WP_HTML_Tag_Processor` — replacing decade-old hacks that wrap `ob_start()` in `template_redirect`.

```php
add_filter( 'wp_template_enhancement_output_buffer', function ( $html ) {
    $p = new WP_HTML_Tag_Processor( $html );
    while ( $p->next_tag( [ 'tag_name' => 'IMG' ] ) ) {
        $p->set_attribute( 'loading', 'lazy' );
    }
    return $p->get_updated_html();
} );
```

Companions:

- `wp_template_enhancement_output_buffer_started` — buffer just opened (set headers)
- `wp_finalized_template_enhancement_output_buffer` — last call before flush; can still send HTTP headers
- `wp_should_output_buffer_template_for_enhancement` — control activation (`__return_true` to force)

## On-demand block CSS for classic themes (6.9)

`should_load_separate_core_block_assets` filter now applies to classic themes (was block-themes-only). Plus `should_load_block_assets_on_demand` (6.8) decouples on-demand asset loading from the legacy split-stylesheets toggle. Pair with `enqueue_empty_block_content_assets( bool, $block_name )` to force assets for hidden blocks where needed.

`styles_inline_size_limit` default raised to 40KB (was 20KB) — more stylesheets get inlined automatically.

`oembed_discovery_links` filter to disable auto-printed oEmbed `<link>` tags (small bytes win for sites that don't expose oEmbed).

## WP-Cron execution moved from `init` to `shutdown` (6.9)

Spawning the loopback request to run cron events used to fire on `init`, blocking TTFB. 6.9 moves it to `shutdown`. **No code change required for most code, but** plugins that manually triggered cron via `do_action( 'init' )` or that benchmarked cron-spawn timing on `init` will see different numbers.

Disable WP-Cron loopback entirely (recommended for production) and run `wp cron event run --due-now` from system cron. See `wp-wpcli-and-ops` skill.

## `supports.visibility` — Hide Blocks default-on (6.9)

New default-on supports flag. Hidden blocks are completely omitted from rendered HTML *and* their scripts/styles are not enqueued. **Real perf win** — a hidden tracking block does not enqueue its analytics script. **But:** if a block has side effects in `render.php` (logging, cache priming, view counting), those side effects no longer fire when hidden.

Toggle off per-block:

```php
add_filter( 'block_type_metadata', function ( $metadata ) {
    if ( 'my-plugin/critical-tracker' === $metadata['name'] ) {
        $metadata['supports']['visibility'] = false;
    }
    return $metadata;
} );
```

## Block-type metadata collection — bulk registration perf (6.7 / 6.8)

`wp_register_block_metadata_collection()` (6.7) and `wp_register_block_types_from_metadata_collection()` (6.8) eliminate per-request `block.json` filesystem reads + JSON parse. Real perf delta in plugins shipping 20+ blocks; compounds across plugins on the same site.

Build with `wp-scripts build --blocks-manifest` to emit `blocks-manifest.php`. Manifest is opcache-friendly. Known issue (April 2025): `wp-scripts start --blocks-manifest` deletes the manifest file in dev mode.

## Image rendering perf

**6.7:**
- `sizes="auto"` for lazy-loaded images — closes a long-standing CLS gap where lazy-loaded responsive images couldn't compute the intrinsic-sizes hint at parse time.
- HEIC uploads transcode to JPEG on upload (PHP-Imagick path).
- AVIF encoding speed bump.

**6.9 frontend perf field guide** documents incremental wins across image priority and `decoding="async"` defaults.

## HTML API: `set_modifiable_text()`, full-document parser, `serialize_token` public

For content rewriting plugins (lazy-load, link rewriting, srcset injection, inline schema), the HTML API is finally fast enough and full-featured enough to replace regex-based rewriters.

- 6.7: `WP_HTML_Processor::create_full_parser( $html )`, `set_modifiable_text()`, `WP_HTML_Processor::normalize()`. Tag Processor scan 3.5–7.5% faster.
- 6.9: `WP_HTML_Processor::serialize_token()` public; new `wp_js_dataset_name()` / `wp_html_custom_data_attribute_name()` helpers.

Pair with the template enhancement output buffer above for whole-page rewriting that doesn't double-parse.

## What did NOT change

Searched and confirmed not changed across 6.7-6.9:

- **HTTP API timeout defaults**: no dev note in window. The 5s default for `wp_remote_get()` is unchanged. If you're hitting external APIs from request-context code, set explicit `timeout` and don't rely on defaults flipping.
- **No `<picture>` element adoption** in core image rendering. View Transitions plugin is the closest adjacent change.
- **No BMP changes**.

## Sources

- [Speculative Loading in 6.8](https://make.wordpress.org/core/2025/03/06/speculative-loading-in-6-8/)
- [WordPress 6.9 Frontend Performance Field Guide](https://make.wordpress.org/core/2025/11/18/wordpress-6-9-frontend-performance-field-guide/)
- [Consistent cache keys for query groups in 6.9](https://make.wordpress.org/core/2025/11/17/consistent-cache-keys-for-query-groups-in-wordpress-6-9/)
- [Streaming block parser in 6.9](https://make.wordpress.org/core/2025/11/19/introducing-the-streaming-block-parser-in-wordpress-6-9/)
- [Block metadata collection in 6.7](https://make.wordpress.org/core/2024/10/17/new-block-type-registration-apis-to-improve-performance-in-wordpress-6-7/)
- [should_load_block_assets_on_demand in 6.8](https://make.wordpress.org/core/2025/03/24/new-filter-should_load_block_assets_on_demand-in-6-8/)
- [HTML API in 6.7](https://make.wordpress.org/core/2024/10/17/updates-to-the-html-api-in-6-7/)
- [HTML API in 6.9](https://make.wordpress.org/core/2025/11/21/updates-to-the-html-api-in-6-9/)
