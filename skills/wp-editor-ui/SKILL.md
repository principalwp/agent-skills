---
name: wp-editor-ui
description: "Block editor UI conventions: component selection, control placement (toolbar vs sidebar), placeholder/loading/error states, theme token consumption, polish patterns (animation, responsive, modern props), and core block pattern matching. Use alongside wp-block-development when building or reviewing block editor interfaces."
compatibility: "Targets WordPress 6.9+ / Gutenberg 22.6+. Codebase-agnostic."
---

# WP Editor UI Conventions

## When to use

Use this skill when:

- Building a new block's `edit.js` / `Edit` component
- Deciding which `@wordpress/components` to use for a block's controls
- Choosing whether a control belongs in toolbar, sidebar, or canvas
- Reviewing block editor UI for consistency with core conventions

This skill complements `wp-block-development` (which covers block structure,
registration, and rendering) with editor UI conventions (which cover how the
block presents itself in the editor).

## Inputs required

- The block's purpose (content display, data entry, media, container, etc.)
- Whether the project has any documented UI conventions (e.g., a project
  catalog of allowed components or theme tokens) — if so, follow it for
  project-specific patterns

## Three-Tier Interface Hierarchy

Every block has exactly three UI layers, in strict priority order:

| Priority | Layer | Location | Rule |
|----------|-------|----------|------|
| 1 | **Content area** | Canvas (inside block) | Direct manipulation. Primary interface. Always visible. |
| 2 | **Block toolbar** | Floating bar above block | Critical controls that can't be inline. Visible when selected. |
| 3 | **Settings sidebar** | Right panel | Advanced/tertiary controls ONLY. Hidden on mobile by default. |

**Hard rule**: If a control is required for basic block usage, it MUST be in
the content area or toolbar — never sidebar-only.

Source: [Block Design Guidelines](https://developer.wordpress.org/block-editor/explanations/user-interface/block-design/)

## Block Model

All custom blocks MUST be **dynamic blocks** (server-rendered via `render.php`
or `render_callback`). Never use static blocks with `save()` markup stored in
post content.

**Why**: Design updates propagate to all instances without re-saving posts.
Important for any site with a large content library where re-saving every
post to roll out a design change is impractical.

This means:
- `save()` returns `null` (or is omitted entirely)
- `render` field in `block.json` points to `render.php`
- `html` support should be `false` (no HTML editing mode)

## Supports-First Approach

Before writing any custom UI controls, check if `block.json` supports can
provide the UI for free. Supports auto-generate core-matching editor controls
with zero custom code.

**Decision order**:
1. Can a `block.json` support handle this? → Use the support
2. Does `@wordpress/block-editor` have a slot/component? → Use it
3. Does `@wordpress/components` have a matching control? → Use it
4. Only then → Build a custom control

See `references/supports-catalog.md` for the full catalog with decision
guidance per support. See `references/polish-patterns.md` for modern
component props, animation conventions, skeleton loading, and responsive
patterns. For editor-chrome UI outside a specific block (document panels,
sidebars, command palette), see `references/editor-extension-points.md`.

### Recommended Minimum Supports

For most custom blocks, start with:

```json
{
  "supports": {
    "anchor": true,
    "html": false,
    "color": {
      "background": true,
      "text": true,
      "link": true
    },
    "spacing": {
      "margin": true,
      "padding": true
    },
    "typography": {
      "fontSize": true,
      "lineHeight": true
    },
    "interactivity": {
      "clientNavigation": true
    }
  }
}
```

Adjust based on block type:
- **Container blocks**: Add `layout`, `spacing.blockGap`, `dimensions.minHeight`
- **Media blocks**: Add `align` (with `["wide", "full"]`), `dimensions.aspectRatio`
- **Text-heavy blocks**: Add `typography.fontFamily`, `typography.textAlign`
- **Decorative blocks**: Add `border`, `shadow`, `background`

Before adding a support, check what the project's `theme.json` already
enables — adding `typography.fontFamily` to a block when the active theme
disables it at the global level produces a control that does nothing.

## Control Placement

### Toolbar (BlockControls)

Put in toolbar:
- Alignment / display mode toggles
- View switching (list/grid)
- Media replacement buttons
- Block-wide structural actions

Toolbar conventions:
- Group related controls in `<ToolbarGroup>` (each group = visual segment)
- Never one segment per single control
- Icon-based only — controls communicate through icons
- Core auto-adds segment 1 (block switcher, drag, mover) and the kebab menu

### Sidebar (InspectorControls)

Put in sidebar:
- Configuration settings (data source, query parameters)
- Display options (show/hide elements, count limits)
- Style adjustments beyond what supports provide

Sidebar conventions:
- Wrap controls in `<PanelBody title="..." initialOpen={true|false}>`
- First PanelBody typically `initialOpen={true}`, others closed
- Use `group` prop to place controls in the correct tab:

| Group | Tab | Use For |
|-------|-----|---------|
| (default) | Settings | Block-specific settings |
| `"styles"` | Styles | Custom style controls |
| `"color"` | Styles → Color | Custom color alongside supports |
| `"typography"` | Styles → Typography | Custom typography alongside supports |
| `"dimensions"` | Styles → Dimensions | Custom spacing/sizing |
| `"border"` | Styles → Border | Custom border controls |
| `"advanced"` | Settings → Advanced | CSS classes, anchors, power-user options |

### Canvas (Content Area)

Put on canvas:
- `RichText` for editable text
- `MediaPlaceholder` for media upload
- Direct manipulation of content structure
- Live preview matching frontend output

## Component Selection Guide

### When to use each @wordpress/components control

| Need | Component | Don't Use |
|------|-----------|-----------|
| Boolean on/off | `ToggleControl` | CheckboxControl (for multi-select) |
| One from small list (3-6) | `RadioControl` | SelectControl (options hidden) |
| One from large list | `SelectControl` | RadioControl (too many visible) |
| One from very large list | `ComboboxControl` | SelectControl (no search) |
| Numeric with visual feedback | `RangeControl` | TextControl with type="number" |
| Free text | `TextControl` | — |
| Multiple tags/tokens | `FormTokenField` | TextControl with comma separation |
| Theme-aware color | `ColorPalette` | ColorPicker (unless custom needed) |
| Action button | `Button` | Raw `<button>` element |
| Loading state | `Spinner` | Custom loading indicator |
| Status message | `Notice` | Custom alert div |
| Setup/empty state | `Placeholder` | Custom empty state div |

**Key conventions**:
- Always provide visible `label` prop — never use placeholder as label
- Use `__next40pxDefaultSize` prop on form controls for modern sizing
- Use `help` prop for clarifying text, not custom `<p>` elements
- Always `import { ... } from '@wordpress/components'` — never use raw HTML
  elements (`<button>`, `<input>`, `<select>`) where a WP component exists

### Block Editor Components (@wordpress/block-editor)

| Component | Purpose | Key Convention |
|-----------|---------|---------------|
| `useBlockProps()` | Block wrapper attributes | REQUIRED on outermost element. ALL supports depend on it. |
| `BlockControls` | Toolbar slot | Wrap toolbar buttons here |
| `InspectorControls` | Sidebar slot | Use `group` prop for tab placement |
| `InspectorAdvancedControls` | Advanced section | Shorthand for `group="advanced"` |
| `RichText` | Inline text editing | Use `RichText.Content` in save (if save needed) |
| `RichTextToolbarButton` | Toolbar button for RichText formats | Used by `registerFormatType()` — NOT block toolbar. See `wp-block-development/references/inline-and-format-types.md` |
| `MediaPlaceholder` | Media upload placeholder | Full drag-drop + library + URL |
| `MediaUpload` | Media library trigger | Render-prop pattern with `open` function |
| `BlockAlignmentControl` | Alignment toolbar | Prefer `supports.align` in block.json instead |

## Block States

Every block must handle these states:

### 1. Placeholder/Setup State
Use when: No sensible default exists and user input is needed before rendering.
Skip when: Good default content can be provided.

```jsx
<div { ...blockProps }>
  <Placeholder
    icon="admin-site"
    label={ __( 'My Block', 'textdomain' ) }
    instructions={ __( 'Configure the data source.', 'textdomain' ) }
  >
    { /* Setup controls */ }
  </Placeholder>
</div>
```

Rules:
- Grey background signals "needs configuration"
- Provide clear `instructions` beyond just `label`
- Include actionable elements (buttons, inputs)
- `blockProps` MUST be on the wrapper even in placeholder state

### 2. Loading State
Use when: Block fetches async data.

```jsx
if ( isLoading ) {
  return (
    <div { ...blockProps }>
      <Placeholder><Spinner /></Placeholder>
    </div>
  );
}
```

### 3. Error State
Use when: Data fetch or configuration fails.

```jsx
if ( error ) {
  return (
    <div { ...blockProps }>
      <Notice status="error" isDismissible={ false }>
        { error }
      </Notice>
    </div>
  );
}
```

**Hard rule**: Never silently swallow errors. Every `catch` block must set
state that renders user-visible feedback.

### 4. Live Preview State
- Must match frontend output as closely as possible
- Show all editable regions with placeholder text
- Selected: may reveal additional inline controls
- Unselected: pure preview, no editing chrome

## Theme Token Consumption

### Determining the approach

Detect what the project uses:

1. **Check for SCSS pipeline**: Look for `.scss` files in the block plugin,
   webpack/postcss config processing SCSS
2. **Check for theme.json custom tokens**: `settings.custom` entries generate
   `--wp--custom--*` CSS vars
3. **Check for preset tokens**: `settings.color.palette`, `settings.typography.fontSizes`,
   `settings.spacing.spacingSizes` generate `--wp--preset--*` CSS vars

### CSS custom property usage (default approach)

```scss
.wp-block-my-plugin-my-block {
  color: var(--wp--preset--color--contrast, currentColor);
  font-size: var(--wp--preset--font-size--medium, 1rem);
  padding: var(--wp--preset--spacing--30, 1rem);
}
```

Rules:
- Always include a fallback value (second `var()` parameter)
- Use preset vars (`--wp--preset--*`) for values from theme.json presets
- Use custom vars (`--wp--custom--*`) for values from theme.json `settings.custom`
- Never hardcode color hex values — always use a preset or custom var

### SCSS with theme tokens (only if project already uses SCSS)

If the project has an existing SCSS build pipeline, follow the project's
established pattern for consuming theme.json tokens. Common approaches:
- SCSS variables mapped from theme.json at build time
- Direct `var()` usage within SCSS (most common)

### When you still need custom CSS

Keep custom styles for:
- Layout structure (`display`, `flex`, `grid`)
- Decorative elements (borders, shadows, icons)
- Interactive states (`:hover`, `:focus`)
- Responsive behavior beyond block supports

If your block only needs font size, color, and spacing — supports may
eliminate the need for a stylesheet entirely.

## Canonical Edit Function Structure

```jsx
import { __ } from '@wordpress/i18n';
import {
  useBlockProps,
  BlockControls,
  InspectorControls,
  RichText,
} from '@wordpress/block-editor';
import {
  PanelBody,
  ToggleControl,
  ToolbarGroup,
  ToolbarButton,
  Placeholder,
  Spinner,
  Notice,
} from '@wordpress/components';

export default function Edit( { attributes, setAttributes } ) {
  const blockProps = useBlockProps();
  const { content, showTitle, dataSource } = attributes;

  // State 1: Placeholder (if block needs setup)
  if ( ! dataSource ) {
    return (
      <div { ...blockProps }>
        <Placeholder
          icon="admin-site"
          label={ __( 'My Block', 'textdomain' ) }
          instructions={ __( 'Select a data source.', 'textdomain' ) }
        >
          { /* Setup controls */ }
        </Placeholder>
      </div>
    );
  }

  // State 2: Loading
  if ( isLoading ) {
    return (
      <div { ...blockProps }>
        <Placeholder><Spinner /></Placeholder>
      </div>
    );
  }

  // State 3: Error
  if ( error ) {
    return (
      <div { ...blockProps }>
        <Notice status="error" isDismissible={ false }>{ error }</Notice>
      </div>
    );
  }

  // State 4: Live preview with controls
  return (
    <div { ...blockProps }>
      <BlockControls>
        <ToolbarGroup>
          <ToolbarButton icon={ grid } label={ __( 'Grid', 'textdomain' ) }
            isActive={ layout === 'grid' }
            onClick={ () => setAttributes( { layout: 'grid' } ) } />
        </ToolbarGroup>
      </BlockControls>

      <InspectorControls>
        <PanelBody title={ __( 'Display', 'textdomain' ) } initialOpen>
          <ToggleControl
            __nextHasNoMarginBottom
            label={ __( 'Show title', 'textdomain' ) }
            checked={ showTitle }
            onChange={ ( val ) => setAttributes( { showTitle: val } ) }
          />
        </PanelBody>
      </InspectorControls>

      <RichText
        tagName="h2"
        value={ content }
        onChange={ ( val ) => setAttributes( { content: val } ) }
        placeholder={ __( 'Enter heading...', 'textdomain' ) }
      />
    </div>
  );
}
```

## Gold-Standard Examples

Complete edit.js + block.json pairs demonstrating correct editor UI patterns.
Each has inline comments explaining WHY each design decision was made.

| Block archetype | Example | Read when |
|---|---|---|
| Data-fetching (CPT, API, external) | `references/example-data-block.md` | Block fetches async data, needs all 4 states, ToolsPanel sidebar |
| Content editing (text, media, CTA) | `references/example-content-block.md` | Block has direct manipulation via RichText, minimal sidebar |
| Container/layout (section, columns) | `references/example-container-block.md` | Block wraps inner blocks via useInnerBlocksProps |

Pick the archetype that matches the block's behavior, read the
corresponding example, then implement.

## Sources

- [Block Design Guidelines](https://developer.wordpress.org/block-editor/explanations/user-interface/block-design/)
- [Block Supports API](https://developer.wordpress.org/block-editor/reference-guides/block-api/block-supports/)
- [Components Reference](https://developer.wordpress.org/block-editor/reference-guides/components/)
- [Inspector Sidebar Groups](https://developer.wordpress.org/news/2023/06/using-block-inspector-sidebar-groups/)
- [WordPress Storybook](https://wordpress.github.io/gutenberg/)
- [10up Gutenberg Best Practices](https://gutenberg.10up.com/)
