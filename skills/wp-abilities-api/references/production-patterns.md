# Abilities API — production patterns and traps

Patterns and gotchas you only learn by running the API in real environments. Each item answers "would a senior engineer catch a bug they wouldn't otherwise catch?" If the answer is no, it isn't here.

## Registration timing — pre-`init` calls silently fail

Calling `wp_register_ability()` or `wp_register_ability_category()` before `init` has fired (e.g. at `plugins_loaded` or top-of-file) does not throw. It triggers `_doing_it_wrong()` and the registration is dropped. You discover it when the REST endpoint 404s or `wp_get_ability()` returns `null`.

```php
add_action( 'wp_abilities_api_categories_init', 'register_my_categories' );
add_action( 'wp_abilities_api_init',            'register_my_abilities' );
```

Categories must exist before any ability that references them. If you ship both in the same plugin, register on the right hooks — do not collapse into one.

Pre-merge code (v0.1–0.3 from the standalone `WordPress/abilities-api` repo) used un-prefixed hook names (`abilities_api_init`). v0.4.0 (the version that landed in 6.9) renamed every hook with a `wp_` prefix and kept no backcompat shim. Old code no-ops silently in 6.9.

## Categories are mandatory and slug-validated more strictly than abilities

Since v0.3.0, every ability requires a registered category. Both PHP and JS reject abilities without one.

Ability names are `namespace/kebab-case`. **Category slugs are stricter** — no slash, lowercase + hyphens only.

```php
wp_register_ability_category( 'data-retrieval', /* ... */ );  // valid

wp_register_ability_category( 'data_retrieval', /* ... */ );  // silently fails — underscore
wp_register_ability_category( 'DataRetrieval',  /* ... */ );  // silently fails — capitals
wp_register_ability_category( 'plugin/data',    /* ... */ );  // silently fails — slash
```

## `permission_callback` is mandatory and quality is not enforced

Since v0.2.0, registration without `permission_callback` fails. The API does not enforce *quality* of the check, however, and the v0.4.0 split made `check_permissions()` no longer call `validate_input()` first. A permission callback that inspects `$input['post_id']` to call `current_user_can( 'edit_post', $input['post_id'] )` is operating on data that the framework has type-checked (when invoked through the REST adapter) but not necessarily existence-checked.

Defense pattern — never trust `$input` keys not in the schema's `properties`, set `additionalProperties => false`, and validate object existence inside the permission callback:

```php
'permission_callback' => function( $input ) {
    if ( empty( $input['post_id'] ) || ! get_post( $input['post_id'] ) ) {
        return new WP_Error( 'not_found', 'Post does not exist', [ 'status' => 404 ] );
    }
    return current_user_can( 'edit_post', (int) $input['post_id'] );
},
```

If you wrap the auto-registered REST route with your own `register_rest_route()` for additional behavior, **do not duplicate the ability's `permission_callback` in the route definition**. The ability's permission check already runs. Use the route callback only for additive, REST-specific checks (audit, rate limiting). Doubling up means override scenarios silently fail when one path returns false and the other true.

## MCP exposure is double-gated

`meta.show_in_rest` controls REST visibility. **MCP visibility is a separate flag**: `meta.mcp.public => true` adds the ability to the *default* MCP server. Many tutorials show only this default-server path, which forces clients through three meta-tools (`mcp-adapter-discover-abilities`, `mcp-adapter-get-ability-info`, `mcp-adapter-execute-ability`) — three hops per call with no schema in the tool list.

Production pattern: build a custom MCP server that exposes specific abilities directly, keeping `meta.mcp.public => false` to keep them out of the default server.

```php
add_action( 'mcp_adapter_init', function ( $adapter ) {
    $adapter->create_server(
        'orders-mcp-server',                      // ID + WP-CLI handle
        'orders-mcp-server',                      // REST namespace
        'mcp',                                    // REST route
        'Orders MCP Server',
        'Curated abilities for order workflows.',
        'v1.0.0',
        [ \WP\MCP\Transport\HttpTransport::class ],
        \WP\MCP\Infrastructure\ErrorHandling\ErrorLogMcpErrorHandler::class,
        \WP\MCP\Infrastructure\Observability\NullMcpObservabilityHandler::class,
        [ 'shop/create-order', 'shop/cancel-order' ], // explicit allowlist
        [], // resources
        []  // prompts
    );
} );
```

Multi-tenant production sites should always run custom servers. The default server exposes everyone's `mcp.public => true` abilities indiscriminately.

Read-only abilities can also be registered as MCP **resources** rather than tools, so models ingest them as context instead of executing them.

## Transports: STDIO uses WP-CLI; HTTP uses the Automattic remote bridge

Local Claude Desktop / Cursor configs cannot speak the WordPress MCP transport directly. There is no native HTTP/SSE inside WordPress.

- **STDIO**: `wp mcp-adapter serve --server={name} --user={admin_user}`. Every locally invoked ability runs as the named user — capability sandbox = whatever that user has.
- **HTTP**: `@automattic/mcp-wordpress-remote` is a Node proxy that translates MCP-over-HTTP to the REST endpoint. Authenticates with application passwords.

There is no first-party OAuth or scoped-token path. For production, create a dedicated WP user with the minimum role required by your exposed abilities. Application passwords are long-lived credentials and inherit everything that user can do.

## Execution is synchronous — wrap long work in Action Scheduler

The framework has no streaming, no async, no progress reporting. `execute()` runs the callback synchronously and the REST adapter waits. Long-running work (image generation, bulk import, AI-assisted content) hits PHP-FPM timeouts, MCP transport timeouts, and blocks workers.

Pattern: kick off via Action Scheduler / wp-background-processing and return a job ID + polling ability.

```php
// Ability 1: starts the job, returns { job_id, status_url }
// Ability 2: polls — readonly: true, idempotent: true, so LLM clients can loop without confirmation prompts
```

## Three return shapes from `execute()` — and they mean different things

```php
$result = $ability->execute( $input );

if ( is_wp_error( $result ) ) {
    // Validation, callback-thrown error, or explicit WP_Error from execute_callback.
} elseif ( null === $result ) {
    // Permission denied, OR invalid callable, OR output validator rejected the result.
    // Use $ability->check_permissions( $input ) to disambiguate.
} else {
    // Array result. Validated against output_schema.
}
```

Treating `null` as "no data" is the bug. Reserve `null` for nothing and return `WP_Error` from `execute_callback` for known failure modes.

## Schema validator is a JSON Schema draft-04 subset

Supported keywords are the WP REST validator set: `type`, `properties`, `additionalProperties`, `required`, `enum`, `items`, `format` (limited), `default`. **Not supported**: `$ref`, `$defs`, `if/then/else`, `oneOf`, `anyOf`. Don't bring complex JSON Schema tooling that relies on draft-07.

LLM constrained-decoders also strip non-structural keywords. `format: "email"`, `pattern`, `minLength` validate server-side but won't constrain the LLM's generation — the agent can produce `"foo"` for an email field and only get rejected at validation time, costing a round-trip.

Practical implication: duplicate critical constraints in the `description` so the LLM sees them.

```php
'email' => [
    'type'        => 'string',
    'format'      => 'email',
    'description' => 'Customer email. MUST be a valid RFC 5322 email; the call will fail validation otherwise.',
],
```

## Extension without forking — `wp_register_ability_args` filter

When a plugin registers an ability with permissive permission semantics, enterprise sites often need to tighten without forking. `wp_register_ability_args` lets you wrap the existing callback and add checks.

```php
add_filter( 'wp_register_ability_args', function ( $args, $name ) {
    if ( 'their-plugin/dangerous-thing' !== $name ) {
        return $args;
    }
    $original = $args['permission_callback'];
    $args['permission_callback'] = static function ( $input = null ) use ( $original ) {
        $base = is_callable( $original ) ? $original( $input ) : true;
        if ( ! $base || is_wp_error( $base ) ) {
            return $base;
        }
        return current_user_can( 'company_extra_cap' );
    };
    return $args;
}, 10, 2 );
```

The same filter can re-label, narrow schemas, force `meta.show_in_rest => false` to remove an ability from REST without unregistering it (so internal PHP callers still work), or graft additive output-schema fields onto third-party abilities.

## Per-ability hooks via custom `WP_Ability` subclass

The `wp_before_execute_ability` / `wp_after_execute_ability` hooks fire for *every* ability — too noisy for fine-grained logging or rate limiting. Subclassing scopes the hook to one ability:

```php
class Order_Create_Ability extends WP_Ability {
    protected function do_execute( $input = null ) {
        $idem = $input['idempotency_key'] ?? null;
        if ( $idem && $cached = get_transient( "order_idem_$idem" ) ) {
            return $cached;
        }
        $result = parent::do_execute( $input );
        if ( $idem && ! is_wp_error( $result ) ) {
            set_transient( "order_idem_$idem", $result, HOUR_IN_SECONDS );
        }
        return $result;
    }
}

wp_register_ability( 'shop/create-order', [
    /* ... */
    'ability_class' => Order_Create_Ability::class,
] );
```

Cleanest place to wedge in caching, idempotency, audit emission, or transactional wrappers before the framework adds them natively.

## Versioning is unsolved — schema breaking changes break agents silently

There is no `version` field on abilities, no deprecation lifecycle, no schema-evolution helpers. If a plugin updates and renames an output property, every chained agent call downstream breaks at the *next* ability — which fails output validation or pulls a missing key — rather than at the source.

Workarounds:

- include a `version` field inside `meta` and check it in callers
- ship two abilities in parallel during deprecation windows (`shop/create-order` + `shop/create-order-v2`); unregister the old via `wp_unregister_ability()` after the window
- use `wp_register_ability_args` to graft additive output fields onto legacy abilities

## Discovery endpoint pagination caps at 100

`GET /wp-abilities/v1/abilities` returns 50 per page (max 100). On a real-world WooCommerce + plugins site, you blow past 100 fast. Naive MCP clients that issue a single discovery call see a partial list and silently miss tools.

```
GET /wp-abilities/v1/abilities?category=commerce&per_page=100&page=1
GET /wp-abilities/v1/abilities?category=commerce&per_page=100&page=2
```

Custom MCP servers bypass this — the explicit allowlist passed to `create_server()` is the source of truth.

## Authentication is required on every Abilities REST endpoint

Unlike core REST (where some routes allow anonymous access), every Abilities REST request requires an authenticated user. There is no "public ability" concept. `permission_callback => '__return_true'` does *not* bypass this — auth is checked before the callback.

Sites that want to expose `get-public-stats`-style abilities anonymously must expose them via a separate non-Abilities REST route, defeating most of the registry's value for that use case.

## Client-side packages: don't conflate `@wordpress/abilities` with `@wordpress/core-abilities`

WP 7.0 ships two distinct packages.

- `@wordpress/abilities` — pure store/registry. No transport, no server dependency. Use this in browser extensions, headless agents, and editor-only workflows.
- `@wordpress/core-abilities` — hydrates the store from `/wp-abilities/v1/`. Assumes a WordPress host.

Registering a client-only ability via `registerAbility()` lives in the browser only — it is not visible to MCP, REST, or other tabs. Useful for editor-local state (current focus, unsaved blocks) but easy to misuse: register "save post" client-side and the LLM running through MCP can't see it.

Default to server-side registration unless the ability *requires* browser-only state.

## Naming reality: `wp-abilities/v1`, not `abilities/v1`

Pre-0.4 prototypes and several Sept/Oct 2025 blog posts used `/wp-json/abilities/v1/...`. The shipped namespace is `/wp-json/wp-abilities/v1/...`. Hardcoded URLs in early MCP clients break.

Endpoints (6.9):

```
GET    /wp-abilities/v1/categories
GET    /wp-abilities/v1/categories/{slug}
GET    /wp-abilities/v1/abilities
GET    /wp-abilities/v1/abilities/{name}
GET|POST|DELETE /wp-abilities/v1/abilities/{name}/run
```

The HTTP verb on `/run` is annotation-derived (see annotations table in `execution-and-annotations.md`).

## Predecessor `wp-feature-api` is dead

`composer require automattic/wp-feature-api` and `@automattic/wp-feature-api` are dead paths. The repo was archived 2025-11-21. Migration to `WordPress/abilities-api` is not 1:1 — function names, registration shape, hook prefixes, REST namespace, category model, and annotation defaults all changed. There is no automated codemod.
