# WordPress gotchas and return-value traps

## Return value traps

These functions return surprising values that defeat naive `if` checks:

| Function | What you expect | What you get |
|----------|----------------|--------------|
| `get_post_meta($id, 'missing', true)` | `null` or `false` | `''` (empty string) |
| `get_post_meta($id, 'missing', false)` | `null` | `[]` (empty array) |
| `is_email('good@example.com')` | `true` | The email string itself |
| `has_filter('hook', $callback)` | `true` | The priority (int) — **priority 0 loosely equals false, use `=== false`** |
| `wp_verify_nonce($nonce, $action)` | `true` | `1` (0-12h old) or `2` (12-24h old); `false` on failure |
| `$wpdb->insert(...)` | Insert ID | Rows affected (`1`) — use `$wpdb->insert_id` for the ID |
| `update_post_meta($id, $key, $val)` | `true` on success | `false` when new value === old value — indistinguishable from failure |
| `add_option('existing_key', $val)` | Updates the option | Does nothing, returns `false` — use `update_option()` to upsert |
| `wp_remote_get(...)` on failure | `false` | `WP_Error` object |

The `update_post_meta` no-op-returns-false trap is especially common. Wrap calls in `update_metadata`-style helpers or check `metadata_exists()` + value comparison if you need to distinguish "no change" from "failed to write".

## Common misconceptions

| Misconception | Reality |
|---------------|---------|
| `is_admin()` checks user role | Checks URL path — returns `true` on `admin-ajax.php` too. Not a security gate. |
| `wp_redirect()` stops execution | Neither `wp_redirect()` nor `wp_safe_redirect()` calls `exit` — you must call it yourself. Code after a redirect keeps running. |
| `wp_update_post()` fires different hooks than create | Implemented via `wp_insert_post()` — same hooks fire on create and update. The `$update` parameter passed to `save_post` distinguishes them. |
| `cron_schedules` filter receives core schedules | Receives an empty array — core schedules are merged AFTER the filter. To add a schedule, return your additions; don't `array_merge` with what you're given. |
| `posts_per_page = 0` returns no posts | Silently forced to `1` — use `-1` for all posts. |
| `get_posts` uses the Reading setting for count | Defaults to `5`, not the Reading setting (which `WP_Query` honors via `posts_per_page`). |

## get_posts vs WP_Query defaults

Three defaults differ and cause silent bugs when developers swap one for the other:

| Parameter | `get_posts` | `WP_Query` |
|-----------|------------|------------|
| `suppress_filters` | `true` | `false` |
| `no_found_rows` | `true` | `false` |
| `ignore_sticky_posts` | `true` | `false` |

Refactoring `get_posts(...)` to `new WP_Query(...)` for "more flexibility" silently turns on `posts_*` filters that were previously bypassed, often producing different results.

Also: sticky posts are only prepended when ALL three conditions are met: `is_home()`, page 1, and `ignore_sticky_posts = false`. A custom homepage replacement or a request for page 2 silently drops them.

## Query details

- `post_type = 'any'` excludes types with `exclude_from_search = true` (e.g., revision, nav_menu_item). If you actually want every post type, build the list explicitly.
- `posts_results` filter fires before sticky/status handling; `the_posts` fires after. Choose based on whether you want to see/modify post statuses other than `publish`.
- `'fields' => 'ids'` and `'fields' => 'id=>parent'` are the only two non-empty valid values for that arg. Other strings are silently ignored and you get full post objects.
- `posts_pre_query` filter short-circuits the DB query by returning a non-null array. The cleanest way to mock results in tests, and the cleanest way to inject pre-computed results when caching at the query level.
- `home_url()` = visitor-facing URL; `site_url()` = WP installation files URL. They differ when WordPress is installed in a subdirectory but served from the root. Mixing them produces broken links and asset loads.

## Hook ordering

Loading order (admin request):

```
muplugins_loaded → plugins_loaded → after_setup_theme → init → wp_loaded
```

Post-save hook order (most specific first):

```
save_post_{$post_type} → save_post → wp_insert_post
```

`the_content` filter priorities:

```
priority 8: block hooks insertion
priority 9: do_blocks
priority 10: wpautop, wptexturize
priority 11: do_shortcode
```

A `the_content` filter at priority < 8 sees raw block markup; > 9 sees rendered HTML; > 10 also sees `wpautop`'d paragraphs. Pick priority based on whether you need to read pre- or post-rendered content.

- `did_action('hook')` returns the number of times fired, not boolean. `if ( did_action(...) )` works for fire/no-fire detection but `=== true` does not.
