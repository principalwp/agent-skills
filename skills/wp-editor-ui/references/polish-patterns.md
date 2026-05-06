# Block Editor Polish Patterns

Patterns that move blocks from "structurally correct" to "feels native."
Use these alongside the main wp-editor-ui conventions and supports-catalog.

All timing values, easing functions, and patterns in this file are sourced
from the Gutenberg codebase on trunk. Source files are cited per section.

## Modern Component Props

WordPress 6.8+ logs deprecation warnings for components using legacy sizing
and margin defaults. All new blocks must opt in to the modern behavior.

Two migration props must be added to all form controls. Without them,
controls use legacy sizing/margins and emit console deprecation warnings.

- `__next40pxDefaultSize` — 40px height instead of 36px. Removal of the
  36px default is targeted for WordPress 7.1.
- `__nextHasNoMarginBottom` — removes legacy bottom margins. On Gutenberg
  trunk (targeting WP 7.0) this is already the default; include it now for
  WP 6.8/6.9 compatibility.

```jsx
{ /* Size-only: controls that accept __next40pxDefaultSize */ }
<TextControl __next40pxDefaultSize label={ __( 'URL' ) } />
{ /* Margin-only: controls that accept __nextHasNoMarginBottom */ }
<ToggleControl __nextHasNoMarginBottom label={ __( 'Show title' ) } />
{ /* Both props: controls that accept both (include both) */ }
<RangeControl __next40pxDefaultSize __nextHasNoMarginBottom label={ __( 'Columns' ) } />
{ /* Wrapper: BaseControl passes margin removal to custom child content */ }
<BaseControl __nextHasNoMarginBottom>
  { /* custom control content */ }
</BaseControl>
```

**`__next40pxDefaultSize`** — fires deprecation via `maybeWarnDeprecated36pxSize`:
`BoxControl`, `ComboboxControl`, `CustomSelectControl`, `FontSizePicker`,
`FormFileUpload`, `FormTokenField`, `InputControl`, `NumberControl`,
`RangeControl`, `SelectControl`, `TextControl`, `TreeSelect`, `UnitControl`.
Also in type definitions: `BorderBoxControl`, `BorderControl`, `Button`,
`ClipboardButton`, `FocalPointPicker`, `QueryControls`, `RadioGroup`,
`SearchControl`, `ToggleGroupControl`.

**`__nextHasNoMarginBottom`** — components with the prop:
`AnglePickerControl`, `BaseControl`, `CheckboxControl`, `ComboboxControl`,
`FocalPointPicker`, `FontSizePicker`, `FormTokenField`, `RadioControl`,
`RangeControl`, `SearchControl`, `SelectControl`, `TextControl`,
`TextareaControl`, `ToggleControl`, `ToggleGroupControl`.

Source: `packages/components/src/utils/deprecated-36px-size.ts` (since 6.8,
removal 7.1).

## ToolsPanel (Modern Sidebar)

New blocks should use `ToolsPanel` + `ToolsPanelItem` instead of `PanelBody`
for sidebar controls. ToolsPanel gives users progressive disclosure — they see
only the controls they've customized, with a menu to add more.

```jsx
import {
  __experimentalToolsPanel as ToolsPanel,
  __experimentalToolsPanelItem as ToolsPanelItem,
} from '@wordpress/components';

<InspectorControls>
  <ToolsPanel
    label={ __( 'Display settings', 'textdomain' ) }
    resetAll={ () => {
      setAttributes( { columns: undefined, gap: undefined } );
    } }
  >
    <ToolsPanelItem
      label={ __( 'Columns', 'textdomain' ) }
      hasValue={ () => columns !== undefined }
      onDeselect={ () => setAttributes( { columns: undefined } ) }
      isShownByDefault
    >
      <RangeControl
        __next40pxDefaultSize
        __nextHasNoMarginBottom
        label={ __( 'Columns', 'textdomain' ) }
        value={ columns }
        onChange={ ( val ) => setAttributes( { columns: val } ) }
        min={ 1 }
        max={ 4 }
      />
    </ToolsPanelItem>

    { /* Conditional item: only renders when columns > 1 */ }
    { columns > 1 && (
      <ToolsPanelItem
        label={ __( 'Column gap', 'textdomain' ) }
        hasValue={ () => gap !== undefined }
        onDeselect={ () => setAttributes( { gap: undefined } ) }
      >
        <UnitControl
          __next40pxDefaultSize
          label={ __( 'Column gap', 'textdomain' ) }
          value={ gap }
          onChange={ ( val ) => setAttributes( { gap: val } ) }
        />
      </ToolsPanelItem>
    ) }
  </ToolsPanel>
</InspectorControls>
```

Note: For clarity, callbacks (`resetAll`, `hasValue`, `onDeselect`) are
inline above. In production, wrap them in `useCallback` to avoid unnecessary
re-renders — the performance-frontend reviewer will flag inline callbacks
passed as props.

When to use ToolsPanel vs PanelBody:
- **ToolsPanel**: New blocks with 3+ optional settings. Cleaner sidebar.
- **PanelBody**: Blocks with 1-2 required settings. Simpler.

Note: `ToolsPanel` and `ToolsPanelItem` are still exported as
`__experimentalToolsPanel`. The API is stable and used throughout core
blocks. Import with the alias pattern shown above.

Source: `packages/components/src/index.ts` — still `__experimental` exports.

## Responsive Editor UI

Use `useViewportMatch()` to adapt block editor UI to viewport size. This
hook returns `true` when the viewport matches the named breakpoint (using
`min-width` by default).

```jsx
import { useViewportMatch } from '@wordpress/compose';
import { useBlockProps } from '@wordpress/block-editor';

export default function Edit( { attributes, setAttributes } ) {
  const blockProps = useBlockProps();
  const isMobile = ! useViewportMatch( 'medium' ); // below 782px

  return (
    <div { ...blockProps }>
      { isMobile ? (
        <CompactMobileLayout { ...attributes } />
      ) : (
        <FullDesktopLayout { ...attributes } />
      ) }
    </div>
  );
}
```

Breakpoints (from `@wordpress/compose` and `packages/base-styles/_breakpoints.scss`):

| Name | Width | Note |
|------|-------|------|
| `xhuge` | 1920px | |
| `huge` | 1440px | |
| `wide` | 1280px | |
| `xlarge` | 1080px | |
| `large` | 960px | Admin sidebar auto-folds |
| `medium` | 782px | Admin bar switches to mobile |
| `small` | 600px | |
| `mobile` | 480px | |

SCSS also has `$break-zoomed-in: 280px` (not available via JS hook).

Common patterns:
- Hide non-essential toolbar buttons on mobile: `{ ! isMobile && <ToolbarButton /> }`
- Switch grid → stack on small viewports
- Reduce placeholder instructions to short text on mobile
- Use `useViewportMatch( 'medium' )` as the primary mobile check — aligns
  with WordPress admin's own responsive breakpoint

Source: `packages/compose/src/hooks/use-viewport-match/index.js`,
`packages/base-styles/_breakpoints.scss`.

## Transition and Animation Conventions

Gutenberg defines transition tokens in JS (`config-values.js`) and uses
per-component SCSS transitions. There is no single global SCSS timing
variable — each component declares its own. The JS tokens are the closest
thing to a canonical reference.

### Transition tokens

From `packages/components/src/utils/config-values.js`:

| Token | Value | Use for |
|-------|-------|---------|
| `transitionDuration` | `200ms` | Default transitions (modal appear, dropdown slide, toggle) |
| `transitionDurationFast` | `160ms` | Faster interactions |
| `transitionDurationFaster` | `120ms` | Quick state changes (alignment matrix) |
| `transitionDurationFastest` | `100ms` | Immediate feedback (button box-shadow, focus, checkbox) |
| `transitionTimingFunction` | `cubic-bezier(0.08, 0.52, 0.52, 1)` | General transitions |
| `transitionTimingFunctionControl` | `cubic-bezier(0.12, 0.8, 0.32, 1)` | Form control interactions |

These tokens are consumed by Emotion (CSS-in-JS) in `@wordpress/components`.
They are NOT available as SCSS variables or CSS custom properties. For custom
block SCSS, reference the raw values directly.

### Real component timing

| Component | Duration | Easing | Source |
|-----------|----------|--------|--------|
| Fade in/out (base mixin) | 80ms | linear | `base-styles/_animations.scss` |
| Dropdown/popover slide | 200ms | `cubic-bezier(0, 0, 0, 1)` | `utils/dropdown-motion.ts` |
| Dropdown/popover fade | 80ms | linear | `utils/dropdown-motion.ts` |
| Appear animation | 100ms | `cubic-bezier(0, 0, 0.2, 1)` | `animate/style.scss` |
| Slide-in animation | 100ms | `cubic-bezier(0, 0, 0.2, 1)` | `animate/style.scss` |
| Modal appear/disappear | 200ms | `cubic-bezier(0.29, 0, 0, 1)` / `cubic-bezier(1, 0, 0.2, 1)` | `modal/style.scss` |
| Button box-shadow | 100ms | linear | `button/style.scss` |
| Button border/bg/color | 50ms | ease-in-out | `button/style.scss` |
| FormToggle track+thumb | 200ms | ease / ease-out | `form-toggle/style.scss` |
| Sidebar slide | 140ms | ease-in-out | `edit-site sidebar/style.scss` |
| Loading pulse | 1600ms | ease-in-out (infinite) | `animate/style.scss` |
| Spinner rotation | 1400ms | linear (infinite) | `spinner/styles.ts` |
| View transitions (root) | 250ms | browser default | `boot/view-transitions.scss` |

### Practical guidance for custom blocks

For custom block transitions, use values from the table above based on the
interaction type:

- **Hover/focus feedback**: 100ms linear (matches button box-shadow)
- **State toggle**: 200ms ease (matches FormToggle)
- **Panel enter/exit**: 100ms `cubic-bezier(0, 0, 0.2, 1)` (matches appear)
- **Content fade**: 80ms linear (matches base fade mixin)

### Safe animation properties

Only animate properties that don't trigger layout recalculation:
- `opacity` — fades (compositor-only, cheapest)
- `transform` — scale, translate, rotate (compositor-only)
- `box-shadow` — focus rings (Gutenberg's primary focus pattern)
- `clip-path` — reveals (triggers paint, not layout; avoid on large elements)
- `filter` — blur, brightness (triggers paint; can be expensive on large
  elements — use sparingly)

Never animate: `width`, `height`, `top`, `left`, `margin`, `padding`,
`font-size`, `border-width`.

### Reduced motion

Gutenberg uses two patterns for `prefers-reduced-motion`. Both are valid:

```scss
// Preferred: wrapper approach (newer Gutenberg files)
// No transition by default — reduced motion users get this
.wp-block-my-plugin-my-block__fade {
  @media not (prefers-reduced-motion) {
    transition: opacity 200ms cubic-bezier(0.08, 0.52, 0.52, 1);
  }
}

// Legacy: override approach (still common in existing code)
.wp-block-my-plugin-my-block__slide {
  transition: transform 100ms cubic-bezier(0, 0, 0.2, 1);
  @media (prefers-reduced-motion: reduce) {
    transition-duration: 0s;
    transition-delay: 0s;
  }
}
```

For animations specifically, use `animation-duration: 1ms` (not `0s`) so
that `animationend` events still fire. Source: `base-styles/_mixins.scss`
`reduce-motion` mixin.

For Interactivity API JS-driven animations:

```js
const prefersReducedMotion = window.matchMedia(
  '(prefers-reduced-motion: reduce)'
).matches;

if ( ! prefersReducedMotion ) {
  element.animate( /* ... */ );
}
```

### When to animate

- **Yes**: State transitions the user triggered (expand/collapse, tab switch,
  show/hide, carousel slide)
- **Yes**: Loading indicators (pulse or spinner)
- **No**: Initial block render — blocks should appear immediately
- **No**: Data updates — new content replaces old content instantly
- **No**: Decorative loops or attention-grabbing animations

## Loading State

Gutenberg does NOT have a skeleton/shimmer component. The core loading
pattern is the `Animate` component's `loading` type — an opacity pulse
(0.5 → 1 → 0.5, 1.6s ease-in-out infinite). For simple cases, use
`<Animate type="loading">` directly. For blocks with a predictable layout
shape, build structural placeholders that mirror the live preview:

```jsx
if ( isLoading ) {
  return (
    <div { ...blockProps }>
      <div className="wp-block-my-plugin-my-block__loading">
        <div className="wp-block-my-plugin-my-block__loading-title" />
        <div className="wp-block-my-plugin-my-block__loading-line" />
        <div className="wp-block-my-plugin-my-block__loading-line" />
      </div>
    </div>
  );
}
```

```scss
// Matches Gutenberg's components-animate__loading (namespaced to block)
@keyframes wp-block-my-plugin-my-block-loading-pulse {
  0% { opacity: 0.5; }
  50% { opacity: 1; }
  100% { opacity: 0.5; }
}

.wp-block-my-plugin-my-block__loading {
  &-title {
    height: 1.5em;
    width: 60%;
  }

  &-line {
    height: 1em;
    width: 100%;
  }

  // Shared styles — currentColor adapts to any theme's text color
  &-title,
  &-line {
    background: currentColor;
    border-radius: 2px;
    margin-bottom: 0.5rem;
    opacity: 0.15;
    @media not (prefers-reduced-motion) {
      animation: wp-block-my-plugin-my-block-loading-pulse 1.6s ease-in-out infinite;
    }
  }
}
```

This approach uses `currentColor` with low opacity instead of hardcoded
colors. It automatically adapts to any theme's text color.

When to use structured loading vs Spinner:
- **Structured loading**: Block has a predictable layout shape (card, list,
  table). The placeholder mirrors that shape.
- **Spinner**: Block layout is unpredictable or the loading state is brief
  (< 300ms). Use `<Placeholder><Spinner /></Placeholder>`.

Source: `packages/components/src/animate/style.scss`,
`packages/components/src/spinner/styles.ts`.

## Hover and Focus States

### Hover (interactive elements)

Gutenberg uses `color-mix()` for hover states. The pattern is a very subtle
tint of the accent color over transparent — NOT a darkening of the base color.

```scss
// Hover: 4% accent tint (matches core Button secondary/tertiary hover)
.wp-block-my-plugin-my-block__interactive {
  &:hover {
    background: color-mix(
      in srgb,
      var(--wp-admin-theme-color, #3858e9) 4%,
      transparent
    );
  }

  &:active {
    background: color-mix(
      in srgb,
      var(--wp-admin-theme-color, #3858e9) 8%,
      transparent
    );
  }
}
```

The consistent Gutenberg pattern is:
- **Hover**: `color-mix(in srgb, <accent> 4%, transparent)` — barely visible tint
- **Active/pressed**: `color-mix(in srgb, <accent> 8%, transparent)` — slightly stronger

This applies to interactive elements in the **editor UI only**. For frontend
render output, use theme.json preset colors instead of `--wp-admin-theme-color`.

Source: `packages/components/src/button/style.scss`,
`packages/components/src/form-token-field/style.scss`.

### Focus-visible (editor UI)

Gutenberg uses `box-shadow` for focus rings (not `outline`). The transparent
`outline` exists as a fallback for Windows High Contrast Mode.

```scss
// Matches Gutenberg's button-style__focus mixin
.wp-block-my-plugin-my-block__interactive:focus-visible {
  box-shadow: 0 0 0 var(--wp-admin-border-width-focus, 2px)
              var(--wp-admin-theme-color, #3858e9);
  outline: 2px solid transparent; // Windows High Contrast Mode
}
```

The outset variant (for elements on colored backgrounds):

```scss
// Matches button-style-outset__focus mixin
.wp-block-my-plugin-my-block__toggle:focus-visible {
  box-shadow: 0 0 0 var(--wp-admin-border-width-focus, 2px) #fff,
              0 0 0 calc(2 * var(--wp-admin-border-width-focus, 2px))
              var(--wp-admin-theme-color, #3858e9);
  outline: 2px solid transparent;
  outline-offset: 2px;
}
```

Rules:
- Use `:focus-visible`, not `:focus` — avoids outlines on mouse click
- Use `box-shadow` for the visible ring, `outline: 2px solid transparent`
  for High Contrast Mode
- `--wp-admin-border-width-focus` is `2px` (1.5px on retina)
- `--wp-admin-theme-color` defaults to `#3858e9` in the modern admin scheme
  (default since WP 5.7; classic scheme uses `#007cba`). All code examples
  in this file use `#3858e9` as the fallback. The component cascade is:
  `var(--wp-components-color-accent, var(--wp-admin-theme-color, #3858e9))`

**Important**: `--wp-admin-theme-color` exists in the editor only. For focus
styles in frontend-rendered output (`render.php` / `style.css` loaded on
the frontend), use a theme.json preset color instead.

Source: `packages/base-styles/_mixins.scss` (`button-style__focus`,
`button-style-outset__focus`, `input-style__focus` mixins),
`packages/components/src/utils/theme-variables.scss`.

### Disabled state

```scss
.wp-block-my-plugin-my-block__button:disabled,
.wp-block-my-plugin-my-block__button[aria-disabled="true"] {
  opacity: 0.6;
  cursor: not-allowed;
}
```

Target both `:disabled` and `[aria-disabled="true"]` — WordPress components
use either pattern depending on the component. Avoid `pointer-events: none`
unless the element truly should not receive any interaction — it prevents
hover tooltips that explain why the element is disabled.

## Sources

All patterns sourced from `github.com/WordPress/gutenberg` trunk branch:

- `packages/components/src/utils/config-values.js` — transition tokens
- `packages/components/src/utils/dropdown-motion.ts` — dropdown/popover timing
- `packages/components/src/utils/deprecated-36px-size.ts` — 40px migration
- `packages/components/src/utils/theme-variables.scss` — accent color cascade
- `packages/base-styles/_animations.scss` — fade in/out mixins
- `packages/base-styles/_mixins.scss` — reduce-motion, focus, input mixins
- `packages/base-styles/_breakpoints.scss` — responsive breakpoints
- `packages/components/src/animate/style.scss` — appear, slide-in, loading
- `packages/components/src/button/style.scss` — button transitions, color-mix
- `packages/components/src/modal/style.scss` — modal appear/disappear
- `packages/components/src/form-toggle/style.scss` — toggle timing
- `packages/components/src/spinner/styles.ts` — spinner rotation
- `packages/compose/src/hooks/use-viewport-match/index.js` — viewport hook
- [WordPress 6.8 Component Updates](https://make.wordpress.org/core/2025/03/25/updates-to-user-interface-components-in-wordpress-6-8/)
- [WordPress 6.7 Component Updates](https://make.wordpress.org/core/2024/10/18/editor-components-updates-in-wordpress-6-7/)
- [10up ToolsPanel Guide](https://gutenberg.10up.com/guides/tools-panel/)

## Third-party API fetch hygiene

Blocks that fetch external APIs (weather, sports scores, affiliate data) in `edit()` need four guardrails on top of the three-state render pattern.

```jsx
import { useEffect, useRef, useState } from '@wordpress/element';
import { useDebounce } from '@wordpress/compose';

export default function Edit( { attributes } ) {
  const { query } = attributes;
  const debouncedQuery = useDebounce( query, 300 );
  const [ state, setState ] = useState( { status: 'idle', data: null, error: null } );
  const abortRef = useRef( null );

  useEffect( () => {
    if ( ! debouncedQuery ) return;
    abortRef.current?.abort();
    const ctrl = new AbortController();
    abortRef.current = ctrl;
    setState( { status: 'loading', data: null, error: null } );

    fetch(
      `https://api.example.com/search?q=${ encodeURIComponent( debouncedQuery ) }`,
      { signal: ctrl.signal }
    )
      .then( ( r ) =>
        r.ok ? r.json() : Promise.reject( new Error( `HTTP ${ r.status }` ) )
      )
      .then( ( data ) => setState( { status: 'ready', data, error: null } ) )
      .catch( ( err ) => {
        if ( err.name === 'AbortError' ) return;
        setState( { status: 'error', data: null, error: err.message } );
      } );

    return () => ctrl.abort();
  }, [ debouncedQuery ] );

  // render by state.status...
}
```

Rules:
- **AbortController on unmount AND on query change** — prevents late responses from overwriting current UI (stale-while-typing bug).
- **Ignore `AbortError` in catch** — it fires on intentional cancellations and should not render as an error state.
- **Debounce** query inputs via `useDebounce` — without it every keystroke hits the network.
- **Never fetch dynamic external data from `save()` or `render.php` per-request** — use a PHP transient cache or a custom binding source. Editor-time fetches are fine; frontend per-request fetches will break under load.
- **Credentials**: never inline API keys in JS. Store in `wp_options` (obfuscated) and proxy through a custom REST route with a real `permission_callback`.

## Meta vs attribute: storage decision

Choosing where block data lives — post meta, block attribute, or taxonomy term — affects query-ability, reuse, and revision behavior.

| Need | Use | Why |
|------|-----|-----|
| Queryable via `WP_Query` (`meta_query`) | Post meta | Attributes live in `post_content`; not indexed |
| Shared across multiple block instances in the same post | Post meta | One source of truth; no sync logic |
| Unique per-block-instance (same block twice with different values) | Attribute | Meta is per-post, not per-block |
| Survives copy-paste between posts | Attribute | Meta stays with the old post |
| Tracked in post revisions by default | Attribute (via `post_content`) — or meta with `revisions_enabled` |
| Editable in Document sidebar alongside title/excerpt | Post meta + `PluginDocumentSettingPanel` |

Combined pattern (testimonial block):
- Quote body → **attribute** (per-instance, edited inline).
- Author name → **post meta** (one author per post, also used by templates and feeds).
- Wire meta into the block via `useEntityProp` in `edit()` and `core/post-meta` binding in `render.php` so the two stay synchronized.
