# Interactivity API internals

## SSR-skipped elements

SVG and MATH elements are skipped during server-side directive processing. Directives inside these elements only activate on the client. So a `data-wp-bind--hidden` on an `<svg>` child won't affect the initial paint — produces a flash if you assumed SSR would handle it.

## Server-rendered vs client-only directives

Server-rendered (processed during SSR):
- `data-wp-text`, `data-wp-class`, `data-wp-style`, `data-wp-bind`, `data-wp-context`, `data-wp-each`

Client-only (no server processing):
- `data-wp-on`, `data-wp-init`, `data-wp-watch`, `data-wp-run`, `data-wp-key`

This determines whether you get a clean first paint or a flash. If the visible state of an element depends on a directive in the client-only list, plan for a placeholder or skeleton until hydration completes.

## Namespace separator

Use `::` to reference a different namespace in directives:

```html
<div data-wp-text="otherPlugin::state.title"></div>
```

Without the prefix, the directive resolves against the closest enclosing `data-wp-interactive` namespace — silently picking up the wrong store when interactive regions are nested.

## PHP closures as derived state

In `wp_interactivity_state()`, PHP closures serve as derived state — they're called during `evaluate()` and tracked for client hydration:

```php
wp_interactivity_state( 'myPlugin', [
    'fullName' => function() {
        $state = wp_interactivity_state( 'myPlugin' );
        return $state['firstName'] . ' ' . $state['lastName'];
    },
] );
```

The closure runs server-side at render time. If your derived value depends on data that arrives after the request boundary (e.g., a deferred REST call), you need a client-side equivalent in JS — the closure won't re-run.

## Client navigation

`add_client_navigation_support_to_script_module()` marks a script module for SPA-style client navigation. This is the correct API — do not manually add attributes.

Router `attachTo` option: Choose which DOM parent receives the router region, preventing conflicting renders when multiple interactive regions exist on the same page.

## Block support object form

```json
{
    "supports": {
        "interactivity": {
            "interactive": true,
            "clientNavigation": true
        }
    }
}
```

Both `interactive` and `clientNavigation` are separate opt-in flags. `clientNavigation: true` without `interactive: true` enqueues the router but no store — useful for blocks that just want SPA-style navigation without state.

## data-wp-ignore

Skip directive processing for an element's children:

```html
<div data-wp-ignore>
    <!-- Directives inside here are not processed -->
</div>
```

Use for embedding third-party widgets or pre-rendered HTML you don't want the directive walker to touch.

## Persisting state across navigations

IAPI state is per-hydration — resets on hard reload. For preferences that should survive (theme mode, sidebar state, dismissed notices), hydrate from cookies or user meta and persist from an action.

```php
// Hydrate per-request so SSR is correct on first paint:
$theme = $_COOKIE['my_theme'] ?? 'light';
wp_interactivity_state( 'myPlugin', [ 'theme' => $theme ] );
```

```js
import { store } from '@wordpress/interactivity';

const { state } = store( 'myPlugin', {
    state: {
        theme: 'light', // overridden by SSR hydration
    },
    actions: {
        toggleTheme() {
            state.theme = state.theme === 'light' ? 'dark' : 'light';
            document.cookie =
                `my_theme=${ state.theme }; path=/; max-age=31536000; SameSite=Lax`;
        },
    },
} );
```

- Use `state` (destructured from `store()`) for global, site-wide preferences. Use `getContext()` only for per-element scoped values declared via `data-wp-context`.

- Cookies: fine for anonymous preferences, PHP-readable on next request for SSR-correct first paint. `SameSite=Lax` minimum; add `Secure` on HTTPS.
- User meta: for logged-in persistence. Post to a REST route with a nonce — `apiFetch` (imported as a module) works inside interactive regions.
- Never persist sensitive state to cookies — unsigned cookies are user-editable.
- Hydrate per-request inside a callback hooked on `wp_enqueue_scripts` or `init` — calling `wp_interactivity_state` too late (after blocks render) won't affect first paint.
- Avoid `localStorage` for SSR-sensitive state: it's client-only, so the server paints the default and the client swaps it in, causing a flash. Cookies + server hydration avoid this.

## Hydrating core blocks via WP_HTML_Tag_Processor

To add IAPI directives to markup produced by a core block — where you can't edit `save()` or `render.php` — filter `render_block` server-side and inject `data-wp-*` via tag processor:

```php
add_filter( 'render_block', function ( $block_content, $block ) {
    if ( $block['blockName'] !== 'core/gallery' ) {
        return $block_content;
    }
    $tp = new WP_HTML_Tag_Processor( $block_content );
    if ( $tp->next_tag( 'figure' ) ) {
        $tp->set_attribute( 'data-wp-interactive', 'myPlugin/slider' );
        $tp->set_attribute(
            'data-wp-context',
            wp_json_encode( [ 'index' => 0 ] )
        );
    }
    while ( $tp->next_tag( 'img' ) ) {
        $tp->set_attribute( 'data-wp-bind--hidden', '!state.isActive' );
    }
    return $tp->get_updated_html();
}, 10, 2 );
```

- The filter only writes markup — enqueue the view module separately via `wp_register_script_module` + `wp_enqueue_script_module`.
- Core blocks with `supports.interactivity.clientNavigation` already load the runtime; otherwise opt in globally or conditionally enqueue `@wordpress/interactivity`.
- `set_attribute` after the cursor has moved past the target tag does nothing silently — always check `next_tag()`'s return value before setting.
- Cross-block state lives in the same store namespace — a slider hydrating core/gallery can share `state` with a custom counter block both declaring `data-wp-interactive="myPlugin/slider"`.
- For SSR directives (`data-wp-text`, `data-wp-class`, etc.), the tag-processor injection runs during `render_block`, so PHP processes them before the server sends HTML — no editor/frontend divergence.
