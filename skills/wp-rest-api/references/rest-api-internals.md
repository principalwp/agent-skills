# REST API internals and gotchas

## Parameter resolution priority

When the same parameter appears in multiple locations, priority (highest wins):

```
JSON body > POST body > GET query string > URL route params > registered defaults
```

A common bug: an endpoint defines a default for `per_page` in its `args`, the URL passes `?per_page=50`, but the client also sends a body with `per_page=10`. The body wins. If you intend URL params to be authoritative, validate at the start of the callback rather than relying on argument resolution.

## Permission callback requirement

Since WP 5.5, omitting `permission_callback` triggers `_doing_it_wrong()` notice. The route still registers and works, but it's flagged. For public endpoints, use `'permission_callback' => '__return_true'` — the explicit declaration is part of the documented contract.

Status codes for failed authorization:
- Logged-in user lacking capability: `403 Forbidden`.
- Logged-out user: `401 Unauthorized`.

Returning the wrong code from a permission_callback (e.g., `WP_Error( 'forbidden', '...', [ 'status' => 401 ] )` for a logged-in user) confuses clients that distinguish "log in" from "you can't do this".

## Authentication filter

`rest_authentication_errors` filter controls auth:
- Return `null` → pass through to next handler
- Return `true` → authenticated
- Return `WP_Error` → denied

Returning anything else (including a string or int) is a silent bug — WordPress treats it as authenticated. Custom auth plugins that return `false` thinking it means "not authenticated" actually let the request through.

## Batch operations

Batch is opt-in per route:

```php
register_rest_route( 'myplugin/v1', '/items', [
    // ...
    'allow_batch' => [ 'v1' => true ],
] );
```

Default max batch size: 25 (filterable via `rest_get_max_batch_size`). A client that sends 100 items in one batch silently drops the last 75 with no error response — clients should respect the documented max.

## Pagination

- Default `per_page`: 10
- Max `per_page`: 100 (enforced server-side; passing `per_page=500` clamps to 100, no error)
- Response headers: `X-WP-Total` and `X-WP-TotalPages`

Custom endpoints that return paginated arrays should set these headers explicitly via `WP_REST_Response::header()` — clients walking pages depend on them.

## Special parameters

- `_embed` — embed linked resources inline (one round-trip instead of many)
- `_envelope` — wraps response in `{body, status, headers}` object (useful when downstream can't see real HTTP headers, e.g., JSONP, some CDNs)
- `_fields` — limit response to specific fields (smaller responses, faster mobile)

## Context values

Three valid context values: `view`, `edit`, `embed`.

- `view` — default. Public-safe rendered fields (e.g., `content.rendered`).
- `edit` — returns unfiltered/raw content (`content.raw`). Requires authenticated user with edit capability.
- `embed` — minimal subset suitable for `_embed` inclusion.

If you need raw content (e.g., a TOC plugin that injects HTML you don't want filtered out), request `?context=edit&_fields=content.raw`. Without `context=edit` you get the rendered, filtered output even when you specify `_fields=content.raw`.

## Dynamic actions

`rest_insert_{$post_type}` fires for both create AND update operations. The 3rd parameter (`$creating`) is boolean — `true` for create, `false` for update. Code that registers this hook expecting "fires only on create" mistakenly runs on every update too.

Post-dispatch filter: `rest_post_dispatch` — runs after every REST response has been built, with the response object as the first argument. Useful for adding global headers, instrumenting all REST calls, or modifying responses based on cross-cutting policy.

## rest_ensure_response

Converts any value to `WP_REST_Response` — use it to normalize callback returns. Especially useful when a callback returns either a value or a `WP_Error`: passing both through `rest_ensure_response()` produces consistent shapes downstream.
