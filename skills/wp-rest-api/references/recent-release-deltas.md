# Recent REST API release deltas (WP 6.7 → 6.9)

REST-specific changes a production codebase should act on. Anchored to Make/Core dev notes for 6.7, 6.8, and 6.9.

## New top-level namespace: `wp-abilities/v1` (6.9)

WP 6.9 ships the Abilities API as its own REST namespace — `wp-abilities/v1`, not `wp/v3/abilities`. Endpoints:

```
GET    /wp-abilities/v1/categories
GET    /wp-abilities/v1/categories/{slug}
GET    /wp-abilities/v1/abilities
GET    /wp-abilities/v1/abilities/{name}
GET|POST|DELETE /wp-abilities/v1/abilities/{name}/run
```

Authentication is required on every endpoint — there is no public-ability concept. HTTP method on `/run` is annotation-derived (readonly → GET, destructive+idempotent → DELETE, else POST). Schema-driven input/output validation runs automatically. See the `wp-abilities-api` skill for full coverage.

## `register_post_type()` gains `embeddable` (6.8)

Decouples embed availability from `public`. Sites can ship CPTs that are publicly accessible but excluded from oEmbed — useful for paid posts or content that shouldn't be syndicated to third-party consumers.

```php
register_post_type( 'paid_article', [
    'public'     => true,
    'embeddable' => false,
] );
```

Defaults to the value of `public`. Existing CPTs get the new behavior implicitly — audit any CPT where you want a dark-launch through oEmbed.

## `rest_menu_read_access` filter — public menu exposure (6.8)

First sanctioned way to surface menu/menu-item/menu-location routes publicly. Headless/decoupled plugins no longer need a custom controller to render menus to logged-out users.

```php
add_filter( 'rest_menu_read_access', function ( $allow, $controller_class ) {
    if ( WP_REST_Menu_Locations_Controller::class === $controller_class ) {
        return true;  // expose locations publicly
    }
    return $allow;
}, 10, 2 );
```

Applies to `wp/v2/menus`, `wp/v2/menu-items`, and `wp/v2/menu-locations`.

## `wp/v2/templates` response: `source` can be `'plugin'` (6.7)

Templates registered by plugins via `register_block_template()` show up in the templates endpoint with `source: 'plugin'`, alongside `'theme'` and `'custom'`. Headless front-ends consuming templates need to handle the new value or templates from plugins will be dropped silently.

## `register_post_meta()` gains `label` (6.7)

Surfaces in the bindings UI and elsewhere. Without `label`, the bindings dropdown shows raw meta keys (`_my_plugin_subtitle`), leaking internal naming.

```php
register_post_meta( 'post', 'subtitle', [
    'type'         => 'string',
    'label'        => __( 'Subtitle', 'my-plugin' ),
    'show_in_rest' => true,
    'single'       => true,
] );
```

## Notes feature reuses `wp/v2/comments` with `comment_type=note` (6.9)

WP 6.9 ships private editorial Notes inside the comments table. Default behavior: `wp/v2/comments` queries **exclude** notes (silent filter). Integrations that mirror comments to third-party systems (analytics, moderation, archives) won't see notes unless they explicitly query for `comment_type=note`.

If you maintain a comment-sync integration, audit it now — silent data loss class of bug.

## `rest_preload_api_request()` trailing-slash fix (6.8)

Pre-6.8: a route registered at `/wp/v2/widgets` and a preload request for `/wp/v2/widgets/` (with slash) silently missed cache. **Plugins shipping their own preloaded REST data via `rest_preload_api_request` should re-test after upgrading** — preload hit rates may change.

## What did NOT change (worth knowing)

These were searched and confirmed not changed across 6.7-6.9, despite expectations:

- **Autosaves/revisions controllers**: no dev-note-level changes.
- **Font collections / families / faces**: no controller-level changes (the API is stable since 6.5).
- **Block bindings sources**: deliberately NOT exposed via REST. Sources stay PHP-side; the REST surface for bindings is the post itself.
- **Streaming block parser** (`WP_Block_Processor`): PHP-only, no REST wrapper.
- Zero REST controller deprecations across the window.

## Production gotchas worth re-emphasizing (not deltas — patterns)

These came up consistently in agency-grade write-ups during the window:

### `permission_callback` must return `WP_Error` for 401, `false` for 403

Returning `false` from `permission_callback` produces a 403 (the user is authenticated but lacks permission). For 401 (unauthenticated), return a `WP_Error` with `'rest_authorization_required'` status `401`. WP's default `rest_authentication_errors` filter handles missing auth — don't second-guess it.

### `register_rest_field()` callbacks run per-record — N+1 hazard

`get_callback` fires for every record in a collection. If it does `get_post_meta()` or a database lookup, you've shipped an N+1. Prime caches on the parent query with `_prime_post_caches()` or `update_post_caches()`, or move the field into the post meta cache so it's already warmed.

### `register_meta` with `show_in_rest`: object/array meta needs an explicit schema

```php
register_post_meta( 'post', 'config', [
    'type'         => 'object',
    'single'       => true,
    'show_in_rest' => [
        'schema' => [
            'type'                 => 'object',
            'properties'           => [
                'enabled' => [ 'type' => 'boolean' ],
                'mode'    => [ 'type' => 'string', 'enum' => [ 'auto', 'manual' ] ],
            ],
            'additionalProperties' => false,
        ],
    ],
] );
```

Without a full schema, you'll get `rest_invalid_stored_value` errors as soon as anyone saves anything more complex than a primitive. Also: the post type must declare `supports['custom-fields']` for meta to round-trip through the editor.

### `_embed=author,wp:term` is N+1-friendly; bare `_embed` is not

Bare `_embed` fetches every linked resource — author, terms, featured media, replies, etc. Scope to what you need:

```
GET /wp/v2/posts?_embed=author,wp:featuredmedia
```

The N+1 is documented in Trac #46249 and not fully fixed across releases. For headless front-ends, this is the difference between "page renders in 200ms" and "page renders in 2s on a real-world post."

### Batch endpoint: 25-item cap, no GET, `require-all-validate` semantics

`POST /batch/v1` runs requests in parallel-ish (sequentially, but in one PHP process). Default cap is 25 — bump via `rest_batch_max_request` filter if your client genuinely needs more. **No GET in batch** (read-only ops aren't supported). `validation: 'require-all-validate'` rejects the entire batch if any request fails validation; the default `'normal'` runs everything and reports per-request results. Custom routes opt in with `'allow_batch' => [ 'v1' => true ]`.

### Application Passwords are long-lived and inherit the user's full capability set

Don't treat them as scoped tokens. There's no scope mechanism — the password gets everything the user can do. For machine-to-machine integrations, create a dedicated user with the minimum role. For per-feature scopes, you need a custom OAuth/JWT layer.

### Multisite: REST is per-site

Each site has its own `/wp-json/` namespace. `switch_to_blog()` inside a permission_callback works but is fragile — you must `restore_current_blog()` on every code path including error returns. Prefer wiring per-site routes with `register_rest_route()` after `switch_to_blog()` rather than runtime switching.

### `rest_pre_dispatch` is your ETag / If-None-Match injection point

```php
add_filter( 'rest_pre_dispatch', function ( $result, $server, $request ) {
    if ( 'GET' !== $request->get_method() ) return $result;
    $etag = my_compute_etag( $request );
    $if_none_match = $request->get_header( 'if-none-match' );
    if ( $if_none_match && trim( $if_none_match, '"' ) === $etag ) {
        return new WP_REST_Response( null, 304, [ 'ETag' => '"' . $etag . '"' ] );
    }
    return $result;
}, 10, 3 );
```

Pair with `rest_post_dispatch` to attach the response ETag header on misses. WP doesn't compute ETags automatically.

### VIP edge: REST has a 60-second default TTL

If you're on WP VIP, GET REST responses are edge-cached for 60s by default. Use `wpcom_vip_rest_read_response_ttl` to adjust per-route. Set the X-VIP-Cache-Auth header for cache key personalization. Authorization-bearing requests bypass cache entirely (which can blow your origin if a high-traffic integration sends `Authorization` it doesn't need).

## Sources

- [WordPress 6.7 Field Guide](https://make.wordpress.org/core/2024/10/23/wordpress-6-7-field-guide/)
- [WordPress 6.8 Field Guide](https://make.wordpress.org/core/2025/03/28/wordpress-6-8-field-guide/)
- [WordPress 6.9 Field Guide](https://make.wordpress.org/core/2025/11/25/wordpress-6-9-field-guide/)
- [Plugin template registration in 6.7](https://make.wordpress.org/core/2024/10/20/new-plugin-template-registration-api-in-wordpress-6-7/)
- [rest_menu_read_access in 6.8](https://make.wordpress.org/core/2025/03/27/new-rest-api-filter-for-exposing-menus-publicly-in-wordpress-6-8/)
- [Misc developer changes in 6.8](https://make.wordpress.org/core/2025/03/25/miscellaneous-developer-changes-in-wordpress-6-8/)
- [Abilities API in 6.9](https://make.wordpress.org/core/2025/11/10/abilities-api-in-wordpress-6-9/)
- [Notes feature in 6.9](https://make.wordpress.org/core/2025/11/15/notes-feature-in-wordpress-6-9/)
- [Trac #46249 — `_embed` N+1](https://core.trac.wordpress.org/ticket/46249)
- [VIP REST API docs](https://docs.wpvip.com/wordpress-on-vip/wordpress-rest-api/)
