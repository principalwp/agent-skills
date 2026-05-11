# Block editor UI release deltas (WP 6.7 → 6.9)

`@wordpress/components`, inspector controls, toolbar, command palette, and Site Editor changes. Anchored to Make/Core editor dev notes for 6.7, 6.8, and 6.9.

## Components — promotions, deprecations, and migration paths

### Stabilized (drop the `__experimental` prefix)

- **Composite** (6.8) — replaces `__unstableComposite`. The accessibility-correct primitive for keyboard-navigable button groups.
- **Navigator family** (6.7/6.8) — `Navigator`, `Navigator.Screen`, `Navigator.Button`, `Navigator.BackButton` replace `__experimentalNavigator*`. `goToParent` / `__experimentalNavigatorToParentButton` aliased to `goBack` / `BackButton`.
- **BorderControl**, **BorderBoxControl**, **BoxControl**, **AlignmentMatrixControl** stabilized through 6.8.
- **CustomSelectControl** rewritten in 6.8 with a new internal — props largely compatible, behavior under styles may differ.

### Deprecated / scheduled removal

- **`__experimentalLinkControl*` family** stabilized in 6.8. `LinkControl`, `LinkControlSearchInput`, `LinkControlSearchResults`, `LinkControlSearchItem` are the canonical names. Imports of the old names spam the console on every editor load.
- **`Navigation`** (block-editor menu UI, distinct from `Navigator`) — slated for removal in WP 7.1.
- **`DimensionControl`** — scheduled for WP 7.0 removal.
- **`RadioGroup`**, **`ButtonGroup`** — soft-deprecated in favor of `ToggleGroupControl` and `Composite`.
- **`SearchControl onClose` prop** — removed in 6.9.
- **`__unstableComposite`** — removed in 6.8 (the stable `Composite` is the replacement).

### Tabs (V2) — replaces `TabPanel`

`Tabs` (formerly `__experimentalTabs`) is the new tab primitive used in inspector tabs, the inserter, and color panels. **`TabPanel` still exists but is on the deprecation slope** — third-party blocks using it should migrate. Different prop shape: `Tabs.TabList` + `Tabs.Tab` + `Tabs.TabPanel` instead of a single `tabs={}` array.

Source: [Editor components updates 6.7](https://make.wordpress.org/core/2024/10/18/editor-components-updates-in-wordpress-6-7/), [Components updates 6.8](https://make.wordpress.org/core/2025/03/25/updates-to-user-interface-components-in-wordpress-6-8/)

## Components — 36px → 40px size deprecation (6.8)

Twenty `@wordpress/components` controls (SelectControl, TextControl, InputControl, RangeControl, NumberControl, FormTokenField, UnitControl, ...) warn at the 36px default. Required for accessibility (touch target). Pass `__next40pxDefaultSize` until the new size becomes default:

```jsx
<TextControl __next40pxDefaultSize value={ x } onChange={ setX } />
```

Companion: Modal close button enlarged 24 → 32px; `headerActions` should declare `size="compact"`.

## ToolsPanel — per-control inline reset buttons (6.8)

`ToolsPanel` historically only offered a kebab "Reset all" — every individual reset went through the popover. 6.8 supplements this with **per-control inline reset buttons** in Color, Shadow, and Duotone panels. The kebab still works.

For your own ToolsPanel, the `panelId`/`resetAllFilter` API matured. Pattern:

```jsx
<ToolsPanel
    label={ __( 'Layout' ) }
    resetAll={ resetAll }
    panelId={ blockClientId }
>
    <ToolsPanelItem
        hasValue={ () => !! attributes.heroPadding }
        label={ __( 'Hero padding' ) }
        onDeselect={ () => setAttributes( { heroPadding: undefined } ) }
        isShownByDefault
        panelId={ blockClientId }
    >
        <BoxControl
            __next40pxDefaultSize
            label={ __( 'Hero padding' ) }
            values={ attributes.heroPadding }
            onChange={ heroPadding => setAttributes( { heroPadding } ) }
        />
    </ToolsPanelItem>
</ToolsPanel>
```

`panelId` ties the optional/reset state together; required for persistence.

## Inspector tabs — Settings, Styles, Advanced via `group` prop

Inspector tabs are routed via the `group` prop on `InspectorControls`:

```jsx
<InspectorControls group="styles">{ /* ... */ }</InspectorControls>
<InspectorControls group="settings">{ /* ... */ }</InspectorControls>
<InspectorControls group="advanced">{ /* ... */ }</InspectorControls>
```

Without `group`, controls land in Settings (the default). Use `styles` for visual treatments (colors, typography, spacing) and `advanced` for HTML class names, anchor IDs, custom CSS.

## Inspector tabs framing nuance

Official core docs define **two sidebar slots** (`InspectorControls`, `InspectorAdvancedControls`) and **one toolbar surface** (`BlockControls`). The "three-tier hierarchy" framing some plugin tutorials use is a heuristic, not core terminology — use `InspectorControls group="advanced"` rather than the legacy `InspectorAdvancedControls` for new work.

## Toolbar deltas

**6.8/6.9:**
- **Cut** added to block kebab.
- **Publish/Save/Trash** repositioned in the editor header.
- **Persistent block inserter** — toggleable across sessions.
- **Transforms blocked under `templateLock: contentOnly`** — child blocks can't be converted to other types.

## Block bindings UI (6.7 + 6.9)

- 6.7: Attributes panel surfaces bindings UI directly (was Inspector-only).
- 6.9: One-click switch source / bind / unbind. `getFieldsList()` on the source returns typed field options. `block_bindings_supported_attributes_{$block_type}` filter narrows or widens which attributes a block exposes for binding.

## Pattern overrides UX rework

Pattern overrides (synced-pattern-with-editable-fields):

- 6.9 expands coverage to `caption` for `core/image`.
- "Allow overrides" UX reworked (Gutenberg PR #60769) — clearer per-attribute toggle.

## Distraction-free / Zoom-Out / Command Palette

**6.7:**
- Zoom-Out gets a vertical toolbar, Shuffle (random pattern swap), and section-style switching inline.

**6.8:**
- Command palette commands `Add New Page`, `Open Site Editor`.

**6.9:**
- Command palette goes admin-wide — accessible from any wp-admin screen, not just the editor.

## Native CSS in components

No documented `@container` adoption in `@wordpress/components` 6.7-6.9 (this is a frequent assumption that does not match primary sources). Component CSS uses a mix of utility classes and CSS variables; container queries are not a public API for component theming.

## Site Editor canvas changes

**6.9 enables iframe preparation:**
- `apiVersion: 3` is mandatory for new blocks; lower versions warn (hard cutover in 7.0). 6.9 is the deprecation runway.
- **Direct drag with live drop preview** in Site Editor (no more drop-target ghosting).
- **Notes** feature — editorial annotations on posts (uses `wp/v2/comments` with `comment_type=note`, excluded from default queries).
- **Hide Blocks** — `supports.visibility` default-on; List View "Hidden" indicator + `Ctrl+Shift+H` / `⌘+Shift+H`.
- **Style Book for classic themes** (issue #41119) — was block-themes-only.

## `setAttributes()` accepts an updater function (6.9)

Eliminates the stale-closure bug class.

```js
// stale-closure trap — `attributes` may be outdated when callback fires
setAttributes( { count: attributes.count + 1 } );

// safe — same shape as React's setState updater
setAttributes( prev => ( { count: prev.count + 1 } ) );
```

## Accessibility deltas

70+ improvements landed in 6.8; 44+ in 6.9. Notable:

- 6.8 reworked focus management on inserter, sidebar tabs, link control.
- 6.9 fixed accordion modeling for the new accordion blocks (WCAG-compliant disclosure pattern).

If you ship custom controls inheriting `@wordpress/components`, audit focus/keyboard behavior after upgrading — fixes occasionally surface contrast or focus-visible regressions in custom themes.

## `var:preset|...` pipe syntax in theme.json vs `var(--wp--preset--...)` in CSS files

Load-bearing distinction. **theme.json values use the pipe syntax**: `"color": { "text": "var:preset|color|primary" }`. **Styles in CSS files use the resolved CSS custom property**: `color: var(--wp--preset--color--primary)`. They look interchangeable; they are not. Hand-coded `var(--wp--preset--color--primary)` *inside theme.json* silently fails parsing; the pipe syntax inside a `.css` file is meaningless.

## Sources

- [Editor components updates 6.7](https://make.wordpress.org/core/2024/10/18/editor-components-updates-in-wordpress-6-7/)
- [Components updates 6.8](https://make.wordpress.org/core/2025/03/25/updates-to-user-interface-components-in-wordpress-6-8/)
- [Misc Block Editor Changes 6.7](https://make.wordpress.org/core/2024/10/20/miscellaneous-block-editor-changes-in-wordpress-6-7/)
- [Misc Block Editor Changes 6.8](https://make.wordpress.org/core/2025/03/25/miscellaneous-block-editor-changes-in-wordpress-6-8/)
- [Misc Editor Changes 6.9](https://make.wordpress.org/core/2025/11/25/miscellaneous-editor-changes-in-wordpress-6-9/)
- [Block Bindings UI 6.7](https://make.wordpress.org/core/2024/10/21/block-bindings-improvements-to-the-editor-experience-in-6-7/)
- [Block Bindings 6.9](https://make.wordpress.org/core/2025/11/12/block-bindings-improvements-in-wordpress-6-9/)
- [Notes feature 6.9](https://make.wordpress.org/core/2025/11/15/notes-feature-in-wordpress-6-9/)
- [Hide Blocks 6.9](https://make.wordpress.org/core/2025/12/01/ability-to-hide-blocks/)
- [Iframe preparation 6.9](https://make.wordpress.org/core/2025/11/12/preparing-the-post-editor-for-full-iframe-integration/)
- [A11y improvements 6.8](https://make.wordpress.org/core/2025/03/25/accessibility-improvements-in-wordpress-6-8/)
- [A11y improvements 6.9](https://make.wordpress.org/core/2025/11/19/accessibility-improvements-in-wordpress-6-9/)
- [Tabs V2 (issue #52997)](https://github.com/WordPress/gutenberg/issues/52997)
- [Style Book classic theme (issue #41119)](https://github.com/WordPress/gutenberg/issues/41119)
