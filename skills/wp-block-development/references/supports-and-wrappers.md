# Supports and wrapper attributes

Use this file when choosing `supports` for a block, styling blocks, or when wrapper styling behaves unexpectedly.

## Wrapper attribute patterns

These are required for all blocks:

- In `edit()`, use `useBlockProps()`.
- In `save()`, use `useBlockProps.save()`.
- In dynamic render PHP, use `get_block_wrapper_attributes()`.

These functions merge classes and styles from block supports, custom classNames, and inline styles set by the user. Skipping them breaks theme.json integration.

## Choosing supports for new blocks

When creating a new block, enable supports that let themes control presentation. The goal: blocks adapt to any theme's design system rather than imposing hardcoded styles.

### Recommended defaults for new blocks

```json
{
  "supports": {
    "html": false,
    "color": {
      "text": true,
      "background": true
    },
    "typography": {
      "fontSize": true,
      "fontFamily": true,
      "lineHeight": true
    },
    "spacing": {
      "margin": true,
      "padding": true
    }
  }
}
```

Not every block needs all of these. A small standalone element might only need `color.text` and `typography.fontSize`. A container block likely needs all of them. (If the element must render truly inline *within* paragraph text, it is not a block — see `references/inline-and-format-types.md`.)

**Why this matters:**
- Themes control visual presentation through `theme.json`. Blocks that hardcode colors, font sizes, or spacing ignore the theme's palette and typography scales.
- End users can customize individual block instances through the editor's sidebar controls. Supports enable this without custom code.
- Block themes and design systems (e.g., WordPress Design System) rely on supports for consistent styling across blocks.

### When to add `align` support

```json
{
  "supports": {
    "align": true,
    "align": [ "wide", "full" ]
  }
}
```

Add `align` when the block is a top-level content element that benefits from width control (hero sections, media, content sections). Skip it for nested blocks where alignment doesn't make sense. (True inline content within paragraphs should use the RichText Format API, not blocks — see `references/inline-and-format-types.md`.)

## CSS custom properties over hardcoded values

Instead of hardcoding colors and sizes in SCSS:

```scss
/* Avoid this */
.wp-block-my-plugin-reading-time {
    color: #666;
    font-size: 0.875em;
}
```

Use CSS custom properties from the theme's preset system:

```scss
/* Prefer this */
.wp-block-my-plugin-reading-time {
    color: var(--wp--preset--color--secondary, currentColor);
    font-size: var(--wp--preset--font-size--small, 0.875em);
}
```

The `var()` fallback (second parameter) provides a sensible default when the theme doesn't define that preset.

### When supports can eliminate SCSS entirely

If your SCSS file only sets:
- Font size → use `supports.typography.fontSize`
- Color → use `supports.color.text`
- Margin/padding → use `supports.spacing`

The block may not need a SCSS file at all. The theme's `theme.json` and user-set block styles handle everything.

### When you still need SCSS

Keep custom SCSS for:
- Layout structure (`display`, `flex`, `grid`)
- Decorative elements (borders, shadows, icons)
- Interactive states (`:hover`, `:focus`)
- Responsive behavior that goes beyond block supports

Even when you keep SCSS, use CSS custom properties for any value that varies by theme (colors, font sizes, spacing scales).

## Upstream references

- Block supports: https://developer.wordpress.org/block-editor/reference-guides/block-api/block-supports/
- `get_block_wrapper_attributes()`: https://developer.wordpress.org/reference/functions/get_block_wrapper_attributes/
- Theme.json and global styles: https://developer.wordpress.org/themes/global-settings-and-styles/
