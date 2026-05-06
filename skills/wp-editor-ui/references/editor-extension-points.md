# Editor chrome extension points

Surfaces for adding UI to the editor itself (document panel, new sidebars, command palette) — distinct from block-level extensions covered in `wp-block-development/references/extension-points.md`.

## SlotFill: document-level panels

Use `@wordpress/plugins` `registerPlugin` + a Slot component to add UI to the editor shell.

| Slot | Where it appears | Typical use |
|------|------------------|-------------|
| `PluginDocumentSettingPanel` | Document sidebar (under Status/Categories) | Custom meta UI, any post type |
| `PluginPostStatusInfo` | "Summary" section of the document sidebar | Inline metadata (word count, link count) |
| `PluginSidebar` | New sidebar accessible via a pinned toolbar icon | Rich panel with its own button |
| `PluginPrePublishPanel` | Pre-publish checklist modal | Gating questions before publish |
| `PluginPostPublishPanel` | Post-publish modal | Share/promotion actions |
| `PluginMoreMenuItem` | Editor's kebab (⋮) menu | Keyboard-shortcut-worthy commands without palette entry |

```jsx
import { registerPlugin } from '@wordpress/plugins';
import { PluginDocumentSettingPanel } from '@wordpress/editor';
import { useEntityProp } from '@wordpress/core-data';
import { TextControl } from '@wordpress/components';
import { useSelect } from '@wordpress/data';

const QuoteAuthorPanel = () => {
    const postType = useSelect(
        ( select ) => select( 'core/editor' ).getCurrentPostType(),
        []
    );
    if ( postType !== 'post' ) return null;

    const [ meta, setMeta ] = useEntityProp( 'postType', postType, 'meta' );
    return (
        <PluginDocumentSettingPanel
            name="my-plugin-quote-author"
            title={ __( 'Quote author' ) }
            className="my-plugin-quote-author"
        >
            <TextControl
                __next40pxDefaultSize
                __nextHasNoMarginBottom
                label={ __( 'Author' ) }
                value={ meta?._quote_author || '' }
                onChange={ ( v ) => setMeta( { ...meta, _quote_author: v } ) }
            />
        </PluginDocumentSettingPanel>
    );
};

registerPlugin( 'my-plugin-quote-author', { render: QuoteAuthorPanel } );
```

Gotchas:
- `name` must be unique across **all** plugins — prefix with your slug.
- `PluginDocumentSettingPanel` moved from `@wordpress/edit-post` → `@wordpress/editor` in WP 6.6. The old import still works but logs deprecation warnings; new code must import from `@wordpress/editor`.
- Gate per-post-type via `getCurrentPostType()` — otherwise the panel appears on every post type including CPTs that shouldn't have it.
- `useEntityProp( 'postType', type, 'meta' )` returns `[value, setValue, fullValue]`. The setter merges onto the meta object (`mergedEdits`) — passing a partial is usually safe, but spreading is explicit.
- Meta must be registered with `show_in_rest => true` and `auth_callback` that permits the current user — otherwise `useEntityProp` silently returns undefined for that key.
- Do NOT import from `@wordpress/edit-post` / `@wordpress/edit-site` directly when the same slot exists in `@wordpress/editor`; the unified import works in both post and site editors.

## Command Palette (`@wordpress/commands`, WP 6.3+)

Registers commands into the Cmd/Ctrl+K palette.

| API | When to use |
|-----|-------------|
| `useCommand({ name, label, icon, callback, context })` | One static command, always available in the given context |
| `useCommandLoader({ name, hook })` | Dynamic set of commands built from a store/selector (e.g., "Open draft: {title}" per recent draft) |

```jsx
import { useCommand } from '@wordpress/commands';
import { plus } from '@wordpress/icons';
import { store as editorStore } from '@wordpress/editor';
import { createBlock } from '@wordpress/blocks';
import { useDispatch } from '@wordpress/data';
import { registerPlugin } from '@wordpress/plugins';

const InsertQuoteCommand = () => {
    const { insertBlock } = useDispatch( editorStore );
    useCommand( {
        name: 'my-plugin/insert-quote',
        label: __( 'Insert quote block' ),
        icon: plus,
        context: 'block-selection-edit',
        callback: ( { close } ) => {
            insertBlock( createBlock( 'core/quote' ) );
            close();
        },
    } );
    return null;
};

registerPlugin( 'my-plugin-insert-quote', { render: InsertQuoteCommand } );
```

Rules:
- `context` values: `'root'` (default, always), `'site-editor'`, `'block-selection-edit'`. Affects when the command surfaces.
- Always call `close()` from the callback after dispatching — leaves the palette dismissed.
- `useCommandLoader`'s `hook` runs on every palette open. Keep it cheap or debounced; don't fire REST requests per keystroke without a cache.
- **Keyboard shortcuts are separate.** `@wordpress/commands` is discovery surface; for key bindings use `@wordpress/keyboard-shortcuts` with `useShortcut()`. A command-palette entry is not auto-bound to a key.
- Labels are the primary match target for the palette's fuzzy search — write them as user-facing verbs ("Insert quote block", not "Quote").

Sources: `@wordpress/commands` package docs (6.3+), `packages/commands/README.md`.
