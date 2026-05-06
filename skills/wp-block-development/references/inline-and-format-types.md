# Inline content and RichText format types

Use this file when a requirement says "inline block," "inline element within a
paragraph," "badge in a sentence," or any content that must flow within existing
paragraph text without line breaks above or below.

## The fundamental constraint

**There are no inline blocks in Gutenberg.** Every block registered via
`registerBlockType()` is a block-level entity. `useBlockProps()` produces a
wrapper with data attributes, selection chrome, and a block toolbar that breaks
inline flow. Even spreading block props onto a `<span>` with `display: inline`
CSS does not change this — the block is still a separate node in the block list,
rendered as a sibling to (not inside) the paragraph.

When a requirement asks for "an inline block," the correct response is one of:

1. A **RichText wrapping format** (for styling or marking up selected text)
2. A **RichText object format** (for inserting a self-contained inline widget)
3. Pushing back on the requirement if neither pattern fits

Do NOT attempt: `display: inline` CSS on a block wrapper, `parent: ['core/paragraph']`,
or `__experimentalInline`. None of these produce true inline rendering within a
paragraph's text content in the editor.

## Decision tree

| Requirement | Correct pattern | Example |
|-------------|----------------|---------|
| Wrap selected text with a tag (highlight, badge around words, custom link) | Wrapping format — `registerFormatType()` + `toggleFormat()` | `core/bold`, `core/text-color` |
| Insert a self-contained widget at cursor (icon, footnote marker, inline data) | Object format — `registerFormatType({ object: true })` + `insertObject()` | `core/footnote`, `core/image` (inline), `core/math` |
| Standalone small content (not inside a paragraph) | Normal block with minimal supports | Small card, badge block between paragraphs |

## Pattern 1: Wrapping format

Wraps a text selection with an inline HTML element. Appears in the RichText
toolbar when the cursor is inside a `RichText` field.

```js
import { registerFormatType, toggleFormat } from '@wordpress/rich-text';
import { RichTextToolbarButton } from '@wordpress/block-editor';

registerFormatType( 'my-plugin/highlight', {
    title: 'Highlight',
    tagName: 'mark',
    className: 'my-highlight',
    // Map format attributes to HTML attributes
    attributes: {
        style: 'style',
    },
    edit({ isActive, value, onChange, onFocus }) {
        return (
            <RichTextToolbarButton
                icon="admin-customizer"
                title="Highlight"
                onClick={ () => {
                    onChange(
                        toggleFormat( value, {
                            type: 'my-plugin/highlight',
                        } )
                    );
                    onFocus();
                } }
                isActive={ isActive }
            />
        );
    },
} );
```

**Edit function receives:** `{ isActive, value, onChange, onFocus }`

**Key APIs:**
- `toggleFormat( value, { type } )` — toggle on/off
- `applyFormat( value, format, startIndex, endIndex )` — apply unconditionally
- `removeFormat( value, formatType )` — remove unconditionally

**Core formats using this pattern:**

| Format | `tagName` | `className` |
|--------|-----------|-------------|
| `core/bold` | `strong` | `null` |
| `core/italic` | `em` | `null` |
| `core/code` | `code` | `null` |
| `core/keyboard` | `kbd` | `null` |
| `core/subscript` | `sub` | `null` |
| `core/superscript` | `sup` | `null` |
| `core/text-color` | `mark` | `has-inline-color` |
| `core/link` | `a` | `null` |
| `core/language` | `bdo` | `null` |

## Pattern 2: Object format (inline widget)

Inserts a self-contained inline object at the cursor position. Uses the Unicode
Object Replacement Character (U+FFFC) in the RichText value, with format
metadata stored in the `replacements` array.

```js
import { registerFormatType, insertObject } from '@wordpress/rich-text';
import { RichTextToolbarButton } from '@wordpress/block-editor';

registerFormatType( 'my-plugin/inline-badge', {
    title: 'Badge',
    tagName: 'span',
    className: 'my-badge',
    object: true,
    contentEditable: false,
    attributes: {
        'data-value': 'data-value',
    },
    edit({ value, onChange, onFocus, isObjectActive, activeObjectAttributes }) {
        return (
            <RichTextToolbarButton
                title="Insert Badge"
                onClick={ () => {
                    onChange(
                        insertObject( value, {
                            type: 'my-plugin/inline-badge',
                            attributes: {
                                'data-value': '42',
                            },
                            innerHTML: '42',
                        } )
                    );
                    onFocus();
                } }
                isActive={ isObjectActive }
            />
        );
    },
} );
```

**Edit function receives (different from wrapping):**
`{ value, onChange, onFocus, isObjectActive, activeObjectAttributes, contentRef }`

**Key API:** `insertObject( value, formatToInsert, startIndex, endIndex )`

**Core formats using this pattern:**

| Format | `tagName` | Technique |
|--------|-----------|-----------|
| `core/footnote` | `sup` | `insertObject()` with `innerHTML: '<a href=...>*</a>'`, `contentEditable: false` |
| `core/image` (inline) | `img` | `insertObject()`, self-closing element |
| `core/math` (6.9+) | `math` | `insertObject()` with `innerHTML` for MathML, `contentEditable: false` |

## Complete `registerFormatType()` settings

```js
registerFormatType( 'namespace/name', {
    // Required
    title: string,
    tagName: string,          // HTML tag: 'span', 'mark', 'img', 'sup', etc.
    className: string | null, // CSS class, or null for bare element

    // Optional
    object: boolean,          // true = replacement object (like <img>)
    contentEditable: boolean, // false = non-editable in editor
    interactive: boolean,     // true = element can receive interactions
    attributes: {},           // Map: format-attr-name -> HTML-attr-name
    keywords: string[],       // Up to 3 search keywords

    // Required
    edit: Function,           // React component for toolbar UI
} );
```

## Build and registration

Format types are JavaScript-only — no `block.json`, no PHP registration.

**File structure** (typical):

```
src/
  formats/
    highlight/
      index.js        // registerFormatType() call
      style.scss      // Frontend + editor styles
  index.js            // import './formats/highlight';
```

**Enqueue:** The format JS must load in the editor. Use `enqueue_block_editor_assets`
or include it in the block plugin's `editorScript` build output. There is no
`block.json` field for format types — they piggyback on the plugin's editor script.

**Frontend styles:** Format output is static HTML stored in post content. The
`className` value is the hook for frontend CSS. Enqueue styles via
`wp_enqueue_style()` on `enqueue_block_assets` (or use the block plugin's
`style` handle if the format is part of a block plugin).

## Limitations and gotchas

### Static HTML only — no server rendering

Format output is serialized directly into post content as HTML. There is no
`render.php` or `render_callback` equivalent. The saved markup is what the
frontend renders.

**Consequence for dynamic data:** A weather badge showing live temperature
cannot use a format type alone. The saved HTML is frozen at publish time.

**Workaround for dynamic data:** Save a `data-*` attribute on the inline
element, then hydrate it on the frontend:

- Use `viewScriptModule` (block-level) or an `enqueue_block_assets` script
  that queries `[data-weather-location]` elements.
- Alternatively, use the Interactivity API if the parent block already uses it.

This is a two-part pattern: the format type provides the inline insertion
mechanism in the editor; a separate frontend script provides the live data.

### className: null conflicts

Only one format can claim a bare element (no class) per tag. `core/bold` owns
bare `<strong>`. If you register a format with `tagName: 'strong'` and
`className: null`, it conflicts with core. Always use a unique `className` for
custom formats.

### innerHTML for non-self-closing object formats

If your object format uses a non-self-closing tag (`<span>`, `<sup>`, `<math>`)
and `object: true`, you must provide `innerHTML` in the replacement object.
Without it, the element renders empty because `object: true` tells the
serializer the tag is self-closing.

```js
// Correct: non-self-closing tag with innerHTML
insertObject( value, {
    type: 'my-plugin/footnote',
    attributes: { 'data-id': id },
    innerHTML: `<a href="#${ id }">*</a>`,
} );

// Wrong: empty element — nothing visible
insertObject( value, {
    type: 'my-plugin/footnote',
    attributes: { 'data-id': id },
} );
```

### No sidebar controls

Format types do not have `InspectorControls`. The only UI surface is the
RichText toolbar (via `RichTextToolbarButton`) and optionally a popover
anchored to the toolbar button. For complex configuration, use a `Popover`
component rendered from the `edit` function.

### Format types only work inside RichText fields

The toolbar button appears only when the cursor is inside a `RichText`
component. If a block uses raw HTML or a non-RichText input, format types
are not available.

### No block supports

Format types do not participate in the `supports` system. They cannot use
`useBlockProps()`, `get_block_wrapper_attributes()`, or block-level color/
typography/spacing controls. Style the format's output element directly via
CSS and `className`.

## The common failure mode

When an AI agent receives a requirement like "build a weather widget that
displays inline within paragraph text," the typical failure sequence is:

1. Agent registers a block with `registerBlockType()`
2. Agent adds `display: inline` or `display: inline-block` CSS
3. Frontend might look inline, but the editor shows the block as a separate
   entity with line breaks above and below
4. Agent tries workarounds: `<span>` wrapper, removing margins, adjusting
   block gap — none fix the fundamental block-level insertion
5. Agent documents the limitation in an ADR and falls back to a standalone block

**The correct approach:** Recognize "inline within paragraph" as a format type
requirement during planning, before any code is written. The decision tree at
the top of this file catches this at spec time.

## Upstream references

- Formatting Toolbar API: https://developer.wordpress.org/block-editor/how-to-guides/format-api/
- RichText Reference: https://developer.wordpress.org/block-editor/reference-guides/richtext/
- GitHub #13666 (inline blocks → closed, use formats): https://github.com/WordPress/gutenberg/issues/13666
- GitHub #40051 (object key documentation): https://github.com/WordPress/gutenberg/issues/40051
- GitHub #18490 (dynamic data in formats): https://github.com/WordPress/gutenberg/issues/18490
