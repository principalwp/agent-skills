# Interactivity API release deltas (WP 6.7 → 6.9)

Concrete deltas in directives, store API, and client navigation that a production codebase must act on. Anchored to Make/Core dev notes for 6.7, 6.8, and 6.9.

## Async-by-default migration: `data-wp-on-async*` and `withSyncEvent` (6.8 → 6.9)

The headline migration. Before 6.8, every `data-wp-on--{event}` action ran synchronously in the event handler — blocking the main thread, hurting INP. The plan: flip to async-by-default. Migration:

**6.8** — `data-wp-on-async--*` and `data-wp-on-async-window--*` / `data-wp-on-async-document--*` directives shipped, marking handlers that explicitly do **not** call `event.preventDefault()` / `event.stopPropagation()` synchronously. Plus the `withSyncEvent()` helper for the (rare) handlers that genuinely need sync access to the event:

```js
import { withSyncEvent } from '@wordpress/interactivity';

actions: {
    submitForm: withSyncEvent( function* ( event ) {
        event.preventDefault();           // sync — needed before any await
        const data = new FormData( event.target );
        const result = yield fetch( '/wp-json/...', { method: 'POST', body: data } );
        // ...
    } ),
}
```

**6.9** — `data-wp-on-async*` directives are deprecated. The plain `data-wp-on--*` directive is now async-by-default and the runtime defers handler execution unless the handler is wrapped with `withSyncEvent()`. **Existing `data-wp-on--click` actions that call `event.preventDefault()` will silently break** unless migrated to `withSyncEvent`.

Source: [Interactivity API in 6.8](https://make.wordpress.org/core/2025/03/26/changes-to-the-interactivity-api-in-wordpress-6-8/), [Interactivity API in 6.9](https://make.wordpress.org/core/2025/11/12/changes-to-the-interactivity-api-in-wordpress-6-9/)

## `data-wp-ignore` deprecated; `---suffix` unique-id directive syntax (6.9)

`data-wp-ignore` broke context inheritance and confused the interactivity-router on client-side navigation. The new triple-dash unique-id syntax lets you attach multiple directives of the same family to one element.

```html
<!-- before — only the last data-wp-watch wins -->
<div data-wp-watch="callbacks.hydrate" data-wp-watch="callbacks.log"></div>

<!-- after — both register independently -->
<div
    data-wp-watch---hydrate="callbacks.hydrate"
    data-wp-watch---log="callbacks.log"
></div>
```

Same applies to `data-wp-bind`, `data-wp-class`, `data-wp-on`, `data-wp-watch`, `data-wp-init`, `data-wp-run`. The suffix is identifier-only (used as a key to dedupe registrations).

New TS helpers in `@wordpress/interactivity`:

```ts
import type { AsyncAction, TypeYield } from '@wordpress/interactivity';

const actions: { fetchUser: AsyncAction<User> } = {
    *fetchUser( id: number ): TypeYield<User> {
        const response = yield fetch( `/users/${ id }` );
        return yield response.json();
    },
};
```

## Client navigation matures: stylesheet + module swap, nested regions, `attachTo` (6.9)

6.9 is the first version where `data-wp-router-region` is actually production-viable for non-trivial sites — earlier versions had two real bugs:

**Stylesheet reconciliation:** Pre-6.9, navigating to a page that loaded different CSS than the current page meant the destination's stylesheets were *not* injected. Result: unstyled content after navigation. 6.9 swaps stylesheets to match the destination page.

**Script module reconciliation:** Same problem for script modules — destination-page modules weren't loaded on navigation. 6.9 swaps modules. Blocks declaring `"supports.interactivity": true` get auto-marked for client-navigation loading. New control: `loadOnClientNavigation: true` flag on `wp_register_script_module()` for selective preload, plus `clientNavigationDisabled: true` on the block to opt a block out entirely.

**Nested router regions:** A region inside another region now navigates independently — useful for tab interfaces or modal flows where the outer page should not change.

**`attachTo` directive:** Renders content (via portal-style attachment) into a different DOM location than where the directive lives. Useful for inspector panels, toasts, modals.

```html
<div data-wp-attach-to="body" data-wp-interactive="my/toaster">
    <!-- rendered as a child of <body>, not here -->
</div>
```

**Full-page client navigation extracted to its own import** in 6.9 — the runtime no longer ships full-page navigation by default. Opt in:

```js
import '@wordpress/interactivity-router';
```

Source: [Interactivity API in 6.9](https://make.wordpress.org/core/2025/11/12/changes-to-the-interactivity-api-in-wordpress-6-9/)

## Server-state hydration shape change (6.7 + 6.9)

**6.7** — `getServerState()` and `getServerContext()` from `@wordpress/interactivity` give read-only proxies that update on navigation. Behavior change: `actions.navigate()` no longer overwrites existing client state/context properties — it only adds new properties. Subscribe via `data-wp-watch` callback that diffs server vs client and writes back to the live store.

```js
import { getServerState } from '@wordpress/interactivity';

callbacks: {
    syncCart: ( { state } ) => {
        const server = getServerState();
        if ( server.cart !== state.cart ) {
            state.cart = server.cart;       // explicit opt-in to overwrite
        }
    },
}
```

**6.9** — server-state full-overwrite returns for the specific case of properties whose server value equals the client value (cache invalidation pattern). If you relied on "navigate never overwrites" as an invariant, audit.

## Store-function expression deprecations (6.8)

Two operator deprecations in directive value strings:

- The `!` (NOT) operator is deprecated. Use a derived state getter instead:

```html
<!-- before -->
<div data-wp-bind--hidden="!state.isVisible"></div>
<!-- after -->
<div data-wp-bind--hidden="state.isHidden"></div>
```

- The `.length` accessor on arrays in directive expressions now triggers reactivity correctly (was a bug in 6.7 that returned stale lengths). No code change required, but be aware that `.length`-driven UI may now update where it didn't before.

## `splitTask` for INP-friendly heavy work (6.6 anchor, still current)

Long-running synchronous work in actions blocks the main thread. `splitTask()` from `@wordpress/interactivity` yields back to the browser between chunks:

```js
import { splitTask } from '@wordpress/interactivity';

actions: {
    *processList() {
        for ( const item of state.bigList ) {
            yield splitTask();    // browser breath
            heavyTransform( item );
        }
    },
}
```

Use whenever a single action loops over more than ~100 items.

## `data-wp-each` getter resolution fix (6.9)

Pre-6.9: `data-wp-each` evaluated the iterable expression once on hydration. If the iterable was a derived getter that depended on other state, updates wouldn't re-run the iteration. Fixed in 6.9 — derived iterables now reactively re-render. **Code that worked around the bug by pre-flattening to a plain array can now use derived state directly.**

Companion gotcha (still current): each rendered item needs a unique `data-wp-key` to avoid reorder bugs. Without `data-wp-key`, removing item 3 from a 5-item list causes items 4 and 5 to lose their state (Preact diffs by index).

## Frontend perf: `fetchpriority` and `in_footer` for script modules (6.9)

Script modules now accept the same perf args as classic scripts. Core auto-applies `fetchpriority="low"` to Interactivity API view modules and the `comment-reply` script.

```php
wp_register_script_module( 'my-block/view', $src, $deps, $version, [
    'fetchpriority' => 'low',
    'in_footer'     => true,
] );

// Mutate later
WP_Script_Modules::set_fetchpriority( $id, 'low' );
WP_Script_Modules::set_in_footer( $id, true );
```

## Production gotchas worth knowing

These came up consistently in agency-grade write-ups during the window:

### Analytics tracking breaks under client navigation

Page-view trackers that fire on `DOMContentLoaded` or `pageshow` do not fire when the interactivity-router swaps content via `actions.navigate()`. Symptom: page views drop ~30-50% on sites that adopt `data-wp-router-region` site-wide.

Fix: subscribe to `state.url` from `@wordpress/interactivity-router` and dispatch a custom analytics event. Pattern:

```js
import { store } from '@wordpress/interactivity';

const router = store( 'core/router' );
let lastUrl = router.state.url;

setInterval( () => {
    if ( router.state.url !== lastUrl ) {
        lastUrl = router.state.url;
        window.dataLayer?.push( { event: 'page_view', url: lastUrl } );
    }
}, 100 );
```

Or better, use a `data-wp-watch` directive that observes `state.url`.

### `data-wp-router-region` requires the element to be present in the SSR'd HTML

If a region is conditionally rendered (e.g. inside a `data-wp-bind--hidden` element that's `true` initially), the navigation won't latch onto it. Either render the region unconditionally (and hide its children) or skip the region wrapper for that block.

### Namespace separator in directive value strings: `::`

Cross-store reads in expressions:

```html
<div data-wp-text="other-namespace::state.message"></div>
```

The `::` separator is parsed; a single `:` is a TypeScript-style annotation hint and silently ignored. Easy typo, hard to spot.

### `data-wp-each` with nested directives needs `data-wp-each-key`

Without an explicit key, Preact uses index — and any re-order or removal corrupts component state. Always:

```html
<template
    data-wp-each---product="state.products"
    data-wp-each-key="context.product.id"
>
    <!-- ... -->
</template>
```

## Sources

- [Interactivity API in 6.7](https://make.wordpress.org/core/2024/10/15/subscribe-to-changes-in-the-interactivity-api-state-and-context-on-client-side-navigation-in-6-7/)
- [Interactivity API in 6.8](https://make.wordpress.org/core/2025/03/26/changes-to-the-interactivity-api-in-wordpress-6-8/)
- [Interactivity API in 6.9](https://make.wordpress.org/core/2025/11/12/changes-to-the-interactivity-api-in-wordpress-6-9/)
- [WordPress 6.9 Frontend Performance Field Guide](https://make.wordpress.org/core/2025/11/18/wordpress-6-9-frontend-performance-field-guide/)
- [Script Modules in 6.7](https://make.wordpress.org/core/2024/10/14/updates-to-script-modules-in-6-7/)
