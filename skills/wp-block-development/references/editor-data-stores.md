# @wordpress/data and @wordpress/core-data store patterns

## @wordpress/data

### Store creation

```js
import { createReduxStore, register } from '@wordpress/data';

const store = createReduxStore( 'my-plugin/store', {
    reducer, // Only required property
    actions,
    selectors,
    resolvers,
    controls,
} );
register( store );
```

`registerStore()` is deprecated — use `createReduxStore()` + `register()`.

### Hooks

| Hook | Purpose |
|------|---------|
| `useSelect` | Read data (uses `useSyncExternalStore` internally) |
| `useDispatch` | Get dispatch function for actions |
| `useSuspenseSelect` | Like `useSelect` but throws promises for unresolved selectors (React Suspense) |

### resolveSelect vs select

- `select(store)` — returns cached data immediately; does NOT wait for resolvers.
- `resolveSelect(store)` — returns promises that wait for resolvers to complete.

Code that calls `select(store).getThing()` immediately after dispatch and gets `undefined` is hitting the resolver-not-yet-run case. Use `resolveSelect` (or `useSelect` with proper deps) instead.

### Thunks (modern pattern)

Thunks replace generator-based controls. Argument object properties (lazy-evaluated getters):

```js
const myAction = () => async ({ dispatch, select, resolveSelect, registry }) => {
    const data = await resolveSelect.getItems();
    dispatch.setItems( data );
};
```

### batch()

Groups multiple dispatches, notifies listeners once:

```js
import { batch } from '@wordpress/data';

batch( () => {
    dispatch.setA( valueA );
    dispatch.setB( valueB );
}); // Single notification
```

Without `batch()`, each dispatch triggers re-render of every subscribed component — a common source of editor jank under bulk operations.

### Invalidation

`invalidateResolution(selectorName, args)` — forces re-fetch on next select.
Also: `invalidateResolutionForStore`, `invalidateResolutionForStoreSelector`.

### AsyncModeProvider

Defers `useSelect` re-renders until browser idle using a priority queue. Useful for reducing jank in complex editor UIs where many components subscribe to the same store.

### Cross-store selectors

```js
import { createRegistrySelector } from '@wordpress/data';

const getRelated = createRegistrySelector(
    ( select ) => ( state, id ) => {
        const item = select( 'my-store' ).getItem( id );
        return select( 'core' ).getEntityRecord( 'postType', 'post', item.postId );
    }
);
```

Use this — not direct `select()` calls inside selectors — when reading from another store, otherwise subscriptions don't update when the foreign store changes.

## @wordpress/core-data

### Store name: `core`

### Entity CRUD hook

```js
const { record, editedRecord, edits, edit, save, hasEdits } = useEntityRecord(
    'postType', // kind
    'post',     // name
    postId      // key
);
```

- `record` — server state
- `editedRecord` — server state + pending local edits merged
- `edits` — only the pending local changes

Reading `record` when you wanted the user's in-progress changes (or vice versa) is a frequent source of "why is the form showing stale data" bugs.

### Entity key

Two-part key: `kind` and `name`. Examples: `('postType', 'post')`, `('taxonomy', 'category')`.

`DEFAULT_ENTITY_KEY` is `'id'`, but overrides exist:
- `postType` / `taxonomy` → `'slug'`
- `theme` → `'stylesheet'`

So `getEntityRecord('postType', 'post', 42)` works but `getEntityRecord('postType', 'post', 'page')` looks up by slug. Mixing the two breaks silently.

### useEntityProp

Returns 3-element tuple: `[value, setValue, fullValue]`.

### Permission checking

```js
const canCreate = useSelect( select =>
    select( 'core' ).canUser( 'create', { kind: 'postType', name: 'post' } )
);
```

Uses an `OPTIONS` HTTP request and checks the `Allow` header. If your custom REST endpoint doesn't respond to `OPTIONS` correctly, `canUser` returns `undefined` indefinitely.

### Transient edits

Edits that don't create undo levels:

```js
dispatch( 'core' ).editEntityRecord( 'postType', 'post', id, { blocks }, { isTransient: true } );
```

### Merged edits

Properties marked as `mergedEdits` deep-merge instead of replacing:

```js
// meta is a mergedEdit — this merges into existing meta, not replaces
editEntityRecord( 'postType', 'post', id, { meta: { myKey: 'value' } } );
```

If you write `editEntityRecord(..., { meta: { myKey: 'value' } })` expecting it to replace meta, your other meta keys survive intact — which is usually what you want, but is surprising the first time.

### getEntityRecord vs getRawEntityRecord

- `getEntityRecord()` — returns server state with edits merged.
- `getRawEntityRecord()` — returns server state only, no local edits.

### Auto-generated convenience selectors

Entity config with `plural` name auto-generates: `get{Plural}`, `save{Name}`, `delete{Name}` methods.

## Post save locking

The `core/editor` store exposes locks that block Save/autosave. Used for editorial gates (minimum word count, required meta, approval workflows).

```js
import { useSelect, useDispatch } from '@wordpress/data';
import { store as editorStore } from '@wordpress/editor';
import { useEffect } from '@wordpress/element';

const MinWordCount = () => {
    const { lockPostSaving, unlockPostSaving } = useDispatch( editorStore );
    const wordCount = useSelect( ( select ) => {
        const blocks = select( 'core/block-editor' ).getBlocks();
        return countWordsInBlocks( blocks ); // user-defined
    }, [] );

    useEffect( () => {
        if ( wordCount < 300 ) {
            lockPostSaving( 'my-plugin/min-words' );
        } else {
            unlockPostSaving( 'my-plugin/min-words' );
        }
        return () => unlockPostSaving( 'my-plugin/min-words' );
    }, [ wordCount, lockPostSaving, unlockPostSaving ] );

    return null;
};
```

- Lock key is a namespaced string. Unlocking with a different key silently leaves the post locked.
- `isPostSavingLocked()` returns `true` if **any** key holds a lock — multiple features can lock simultaneously.
- Autosave has its own pair: `lockPostAutosaving` / `unlockPostAutosaving`. Use when manual save should be allowed but autosave disabled.
- A locked state **disables** the Save button; it doesn't hide it. Surface the reason via `PluginPostStatusInfo` or a `Notice` so editors understand why.
- Always clean up via the effect's return — otherwise navigating to another post leaves the next post locked.
