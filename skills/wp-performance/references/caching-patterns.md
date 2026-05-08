# Caching patterns and gotchas

## Caching false/falsey values

The `$found` parameter is the only way to distinguish a cached `false` from a cache miss:

```php
$value = wp_cache_get( 'my_key', 'my_group', false, $found );
if ( ! $found ) {
    // Cache miss — compute and store.
    $value = expensive_computation();
    wp_cache_set( 'my_key', $value, 'my_group' );
}
// $value may legitimately be false, 0, '', null.
```

Without `$found`, code like `if ( false === $value )` will recompute on every request when the real value is `false` — silently turns the "cache" into "always run the expensive computation". Same trap applies to options: `get_option('foo')` returning `false` could mean "doesn't exist" or "the stored value is literally false".

## Salted caching (6.9+)

`wp_cache_get_salted()` / `wp_cache_set_salted()` embed the `last_changed` token into stored values, so stale entries are transparently rejected without explicit cache deletion:

```php
// Read — returns false on miss OR on stale salt.
$titles = wp_cache_get_salted( 'book_titles', 'demo-books' );

if ( false === $titles ) {
    $query  = new WP_Query( [ 'post_type' => 'book', 'post_status' => 'publish' ] );
    $titles = wp_list_pluck( $query->posts, 'post_title' );
    wp_cache_set_salted( 'book_titles', $titles, 'demo-books' );
}
```

Invalidation happens automatically when `wp_cache_set_posts_last_changed()` updates the `last_changed` token (called by `clean_post_cache()`). No explicit `wp_cache_delete()` needed — solves the "I forgot to invalidate after `wp_update_post`" class of stale-cache bugs.

## Batch cache operations

Use `wp_cache_get_multiple()` / `wp_cache_set_multiple()` to reduce round-trips:

```php
$keys   = [ 'option_a', 'option_b', 'option_c' ];
$cached = wp_cache_get_multiple( $keys, 'my-options' );

$to_fill = [];
foreach ( $cached as $key => $value ) {
    if ( false === $value ) {
        $to_fill[ $key ] = get_option( $key );
    }
}
if ( ! empty( $to_fill ) ) {
    wp_cache_set_multiple( $to_fill, 'my-options' );
}
```

Looping `wp_cache_get()` 50 times produces 50 round-trips to the cache backend (Redis, Memcached). `get_multiple` is one round-trip.

## wp_cache_add vs wp_cache_set

- `wp_cache_add()` — fails silently if key already exists (returns `false`).
- `wp_cache_set()` — always overwrites.

Code that uses `wp_cache_add` thinking it's "set if not present, otherwise overwrite" silently fails to update — the stale value persists for the rest of the request. If you want set-or-update, use `wp_cache_set`.

## Non-persistent core groups

These cache groups are non-persistent — data is lost between requests even with an external object cache:

- `counts`, `plugins`, `theme_json`

Code that caches expensive results into one of these groups appears to work in test (single request) but rebuilds on every production request. Don't reuse core's group names; pick your own.

## Transient gotchas

- `set_transient($name, $value, 0)` — expiration of `0` means **never expires**, AND the option is autoloaded. Setting `0` thinking it means "expire immediately" autoloads the value forever.
- Max transient name length: **172 characters** (site transients: 167). Longer names are silently truncated, which means two distinct logical keys can collide on storage.
