# Block extension points

Three ways to extend blocks without forking them: **variations** (alternate presets of an existing block), **filter hooks** (cross-block attribute/UI injection), **transforms** (conversion rules). Use when building on core blocks or shipping reusable editor behavior.

## Block variations (`registerBlockVariation`)

Register JS-side alternatives to an existing block type — appears in inserter (or transform/block scopes) with pre-set attributes and inner blocks.

```js
import { registerBlockVariation } from '@wordpress/blocks';

registerBlockVariation( 'core/group', {
    name: 'my-plugin/card',
    title: __( 'Card' ),
    description: __( 'A bordered card.' ),
    attributes: { className: 'is-style-card', layout: { type: 'constrained' } },
    innerBlocks: [
        [ 'core/heading', { level: 3, placeholder: __( 'Title' ) } ],
        [ 'core/paragraph', { placeholder: __( 'Body' ) } ],
    ],
    scope: [ 'inserter', 'transform' ],
    isActive: [ 'className' ],
} );
```

- `scope`: `'inserter' | 'transform' | 'block'`. Default is all three. `'block'` means selectable only via the block-switcher in the block toolbar.
- `isActive`: array of attribute names OR a function `(blockAttributes, variationAttributes) => boolean`. Controls which variation appears "selected" in UI. Missing/incorrect `isActive` → variation never shows as active even when matched.
- If two variations share overlapping `isActive` attributes, the first registered wins silently.

### Variations + bindings (preset binding source)

Set `attributes.metadata.bindings` on the variation to ship a binding-ready block the editor picks in one click — no hand-wiring via the metadata panel:

```js
registerBlockVariation( 'core/paragraph', {
    name: 'my-plugin/post-byline',
    title: __( 'Post byline' ),
    attributes: {
        metadata: {
            bindings: {
                content: { source: 'core/post-meta', args: { key: '_byline' } },
            },
        },
    },
    isActive: ( blockAttrs ) =>
        blockAttrs?.metadata?.bindings?.content?.args?.key === '_byline',
} );
```

Gotcha: `isActive` as an array (`[ 'metadata.bindings.content.args.key' ]`) does NOT support deep attribute paths — must be a function when the discriminator is nested.

### PHP-side variations

Use the `variations` key in `block.json`, or the `block_type_variations` filter for dynamic variation sets. JS registration wins when both exist.

Unregister: `wp.blocks.unregisterBlockVariation( 'core/group', 'my-plugin/card' )` — run on `domReady` after the target variation is registered.

## Block filter hooks

Four canonical `addFilter` extension points for touching blocks without editing their source.

```js
import { addFilter } from '@wordpress/hooks';
import { createHigherOrderComponent } from '@wordpress/compose';
```

| Hook | Runs on | Typical use |
|------|---------|-------------|
| `blocks.registerBlockType` | Block settings at registration | Inject attributes/supports into target blocks |
| `editor.BlockEdit` | Edit component (HOC) | Add controls/panels to the inspector of target blocks |
| `editor.BlockListBlock` | Block as rendered in the canvas list | Add wrapper className / visual treatment |
| `blocks.getSaveElement` | Save-time React element | Modify static-saved markup (static blocks only) |

### Injecting an attribute + control into every block

```js
addFilter(
    'blocks.registerBlockType',
    'my-plugin/add-note-attr',
    ( settings ) => ( {
        ...settings,
        attributes: {
            ...settings.attributes,
            myPluginNote: { type: 'string', default: '' },
        },
    } )
);

const withNoteControl = createHigherOrderComponent( ( BlockEdit ) => ( props ) => (
    <>
        <BlockEdit { ...props } />
        <InspectorControls>
            <PanelBody title={ __( 'Editorial' ) }>
                <TextareaControl
                    __nextHasNoMarginBottom
                    label={ __( 'Note' ) }
                    value={ props.attributes.myPluginNote }
                    onChange={ ( v ) => props.setAttributes( { myPluginNote: v } ) }
                />
            </PanelBody>
        </InspectorControls>
    </>
), 'withNoteControl' );

addFilter( 'editor.BlockEdit', 'my-plugin/add-note-control', withNoteControl );
```

Gotchas:
- `editor.BlockEdit` expects a component back — use `createHigherOrderComponent` so React DevTools preserves a sensible display name.
- To target a subset, check `name` in the callback and early-return the original component/settings.
- Adding attributes retroactively requires a **deprecation** on target blocks — existing saved markup has no attribute, so WP sees "missing" not "default." Either scope the filter to newly-inserted blocks, or register a deprecation that normalizes the attribute.
- `blocks.getSaveElement` only applies to static blocks. For dynamic blocks, filter PHP side via `render_block` instead.
- Priorities run in registration order — the fourth arg to `addFilter` is a numeric priority when needed.

### PHP-side equivalents for dynamic blocks

`render_block`, `render_block_{namespace}_{slug}`, `render_block_data`, `render_block_context`, `block_type_metadata`, `block_type_metadata_settings`.

## Block transforms

Conversion rules declared on a block's registration under `transforms: { from: [...], to: [...] }`. Powers the block switcher, enter-triggered transforms, paste handling, and file drops.

```js
registerBlockType( 'my-plugin/callout', {
    // ...
    transforms: {
        from: [
            {
                type: 'block',
                blocks: [ 'core/paragraph' ],
                transform: ( { content } ) => createBlock( 'my-plugin/callout', { content } ),
            },
            {
                type: 'enter',
                regExp: /^\/callout$/,
                transform: () => createBlock( 'my-plugin/callout' ),
            },
            {
                type: 'raw',
                isMatch: ( node ) =>
                    node.nodeName === 'BLOCKQUOTE' && node.classList.contains( 'callout' ),
                schema: () => ( {
                    blockquote: {
                        classes: [ 'callout' ],
                        children: { p: { children: getPhrasingContentSchema() } },
                    },
                } ),
                transform: ( node ) =>
                    createBlock( 'my-plugin/callout', { content: node.innerHTML } ),
            },
        ],
        to: [
            {
                type: 'block',
                blocks: [ 'core/paragraph' ],
                transform: ( { content } ) => createBlock( 'core/paragraph', { content } ),
            },
        ],
    },
} );
```

Transform `type` values:

| Type | Trigger |
|------|---------|
| `block` | User invokes the block-switcher in the toolbar |
| `enter` | User types a pattern and presses Enter (supports `regExp`) |
| `raw` | Pasted HTML matches `isMatch` + `schema` |
| `shortcode` | Legacy shortcode conversion during content migration |
| `files` | Files dropped onto the editor (MIME-matched via `isMatch`) |
| `prefix` | Markdown-like prefix (e.g., `> ` → quote) |

Rules:
- First matching transform wins within a type. Use `priority` (default 10) to order against core or other plugin transforms.
- Return `null` from `transform` to bail — the block is left as-is.
- `isMultiBlock: true` — transform accepts an array of source blocks (e.g., merge three paragraphs into one quote).
- `raw` schema format matches `@wordpress/dom/phrasing-content` — start from `getPhrasingContentSchema()` and narrow.
- Transforms run in the editor only. The server sees the output of the transform as normal block markup — no PHP side needed.
