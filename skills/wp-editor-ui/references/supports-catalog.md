# Block Supports Catalog

Full catalog of `block.json` supports as of WordPress 6.9. Use this to decide
which supports to enable for a new block.

## How Supports Work

1. Declare in `block.json` `"supports"` object
2. WordPress auto-registers attributes and generates editor UI
3. CSS classes/inline styles applied via `useBlockProps()` (edit) /
   `get_block_wrapper_attributes()` (PHP render)
4. Theme.json `settings` control which options users see

**ALL supports require `useBlockProps()` on the wrapper element.**

## Quick Decision: Which Supports for My Block?

| Block Type | Recommended Supports |
|------------|---------------------|
| **Text content** (callout, quote, testimonial) | `color.text`, `color.background`, `typography.fontSize`, `typography.lineHeight`, `typography.textAlign`, `spacing.margin`, `spacing.padding`, `anchor` |
| **Container/section** (group, wrapper, section) | All of above + `layout`, `spacing.blockGap`, `dimensions.minHeight`, `align`, `background` |
| **Media** (image wrapper, video, hero) | `align` (with `["wide","full"]`), `dimensions.aspectRatio`, `spacing.margin`, `border`, `shadow` |
| **Data display** (query results, feed, list) | `color.text`, `color.background`, `typography.fontSize`, `spacing.margin`, `spacing.padding` |
| **Small standalone** (badge, tag, label) | `color.text`, `color.background`, `typography.fontSize` only. **If it must render inside paragraph text, use a RichText format type instead of a block** — see `wp-block-development/references/inline-and-format-types.md` |
| **Interactive** (carousel, accordion, tabs) | Base supports + `interactivity: true` or `{ clientNavigation: true }` |

Always also set: `"html": false`, `"anchor": true`, `"interactivity": { "clientNavigation": true }`.

## Full Reference

### Structural Supports

#### align
- **Type**: `boolean | string[]`
- **Default**: `false`
- **Editor UI**: Alignment toolbar buttons (left, center, right, wide, full)
- **Attribute**: `align`
- **theme.json**: Requires `settings.layout.contentSize`/`wideSize` for wide/full
- **Use when**: Block is a top-level content element benefiting from width control
- **Array form**: `["wide", "full"]` limits available options

#### alignWide
- **Type**: `boolean`
- **Default**: `true`
- **Editor UI**: None (controls availability of wide/full in align)
- **Use when**: Set `false` to remove wide/full options

#### allowedBlocks
- **Type**: `boolean`
- **Default**: `false`
- **Editor UI**: Allowed blocks selection in sidebar (container blocks)
- **Requires**: `useInnerBlocksProps` integration
- **Since**: WordPress 6.9

#### layout
- **Type**: `object`
- **Default**: `null`
- **Editor UI**: Content width, justification, orientation controls in sidebar
- **Sub-properties**: `default`, `allowSwitching`, `allowEditing`, `allowInheriting`, `allowSizingOnChildren`, `allowVerticalAlignment`, `allowJustification`, `allowOrientation`, `allowWrap`, `allowCustomContentAndWideSize`
- **Use when**: Block is a container with inner blocks needing layout control

### Appearance Supports

#### color
- **Type**: `object`
- **Default**: `null`
- **Editor UI**: Color panel in Styles tab with palette swatches
- **Sub-properties**:

| Sub-property | Default | UI | Attribute |
|---|---|---|---|
| `background` | `true` | Background color picker | `backgroundColor`, `style.color.background` |
| `text` | `true` | Text color picker | `textColor`, `style.color.text` |
| `gradients` | `false` | Gradient picker | `gradient`, `style.color.gradient` |
| `link` | `false` | Link color picker | `style.elements.link.color.text` |
| `heading` | `false` | Heading color picker (6.5+) | `style.elements.heading.color.text` |
| `button` | `false` | Button color picker (6.5+) | `style.elements.button.color.text` |
| `enableContrastChecker` | `true` | Contrast a11y check | — |

- **CSS classes**: `has-{slug}-background-color`, `has-{slug}-color`, `has-text-color`, `has-background`
- **theme.json**: `settings.color.palette`, `settings.color.gradients`, `settings.color.custom`, `settings.color.customGradient`

#### typography
- **Type**: `object`
- **Default**: `null`
- **Editor UI**: Typography panel in Styles tab

| Sub-property | Default | UI |
|---|---|---|
| `fontSize` | `false` | Font size picker (presets + custom) |
| `fontFamily` | `false` | Font family selector |
| `lineHeight` | `false` | Line height control |
| `fontWeight` | `false` | Font weight selector |
| `fontStyle` | `false` | Font style (normal/italic) |
| `textDecoration` | `false` | Underline/strikethrough |
| `textTransform` | `false` | Uppercase/lowercase/capitalize |
| `letterSpacing` | `false` | Letter spacing control |
| `textAlign` | `false` | Text alignment toolbar buttons |
| `writingMode` | `false` | Writing direction |

- **Attributes**: `fontSize`, `style.typography.*`
- **CSS classes**: `has-{slug}-font-size`
- **theme.json**: `settings.typography.fontSizes`, `settings.typography.fontFamilies`, `settings.typography.customFontSize`

#### spacing
- **Type**: `object`
- **Default**: `null`
- **Editor UI**: Spacing controls in Dimensions panel (Styles tab)

| Sub-property | Default | UI |
|---|---|---|
| `margin` | `false` | Margin controls (all sides or individual) |
| `padding` | `false` | Padding controls |
| `blockGap` | `false` | Gap between child blocks |

- **Array form**: `["top", "bottom"]` limits which sides are configurable
- **Attribute**: `style.spacing`
- **theme.json**: `settings.spacing.spacingSizes`, `settings.spacing.units`
- **Note**: `blockGap` requires the block to also support `layout`

#### border
- **Type**: `object`
- **Default**: `null`
- **Editor UI**: Border & Shadow panel in Styles tab

| Sub-property | Default | UI |
|---|---|---|
| `color` | `false` | Border color picker |
| `radius` | `false` | Border radius control |
| `style` | `false` | Border style (solid, dashed, dotted) |
| `width` | `false` | Border width control |

- **Attributes**: `style.border`, `borderColor`
- **theme.json**: `settings.border.color`, `settings.border.radius`, `settings.border.style`, `settings.border.width`

#### shadow
- **Type**: `boolean`
- **Default**: `false`
- **Editor UI**: Shadow picker in Border & Shadow panel (Styles tab)
- **Attribute**: `style.shadow`
- **theme.json**: `settings.shadow.presets`

#### background
- **Type**: `object`
- **Default**: `null`
- **Since**: WordPress 6.5

| Sub-property | Default | UI |
|---|---|---|
| `backgroundImage` | `false` | Image picker in Styles tab |
| `backgroundSize` | `false` | Size controls (cover, contain, custom) |

- **Attribute**: `style.background`

#### dimensions
- **Type**: `object`
- **Default**: `null`
- **Editor UI**: Controls in Dimensions panel (Styles tab)

| Sub-property | Default | UI |
|---|---|---|
| `aspectRatio` | `false` | Aspect ratio dropdown |
| `minHeight` | `false` | Minimum height input |

- **Attribute**: `style.dimensions`
- **theme.json**: `settings.dimensions.aspectRatios` (6.6+)

#### filter
- **Type**: `object`
- **Sub-property**: `duotone` (boolean, default false) — duotone color filter
- **Replaces**: deprecated `color.__experimentalDuotone`

### Behavioral Supports

#### anchor
- **Type**: `boolean`
- **Default**: `false`
- **Editor UI**: "HTML Anchor" text field in Advanced section + "Copy link" button
- **Attribute**: `anchor`

#### className
- **Type**: `boolean`
- **Default**: `true`
- **Purpose**: Auto-adds `.wp-block-{namespace}-{name}` class to wrapper
- **No additional UI**: Automatic

#### customClassName
- **Type**: `boolean`
- **Default**: `true`
- **Editor UI**: "Additional CSS class(es)" text field in Advanced section
- **Attribute**: `className`

#### html
- **Type**: `boolean`
- **Default**: `true`
- **Purpose**: HTML editing mode toggle
- **Set `false`**: Always for dynamic blocks (which all custom blocks should be)

#### inserter
- **Type**: `boolean`
- **Default**: `true`
- **Purpose**: Block appears in block inserter
- **Set `false`**: For blocks only inserted programmatically (inner blocks of a parent)

#### interactivity
- **Type**: `boolean | object`
- **Default**: `false`
- **Sub-property**: `clientNavigation` (boolean) — supports client-side navigation
- **No sidebar UI**: Enables Interactivity API directive processing

#### lock
- **Type**: `boolean`
- **Default**: `true`
- **Purpose**: Block locking UI (prevent move/remove)

#### multiple
- **Type**: `boolean`
- **Default**: `true`
- **Purpose**: Allow multiple instances per post
- **Set `false`**: For singleton blocks

#### position
- **Type**: `object`
- **Default**: `null`
- **Sub-properties**: `sticky` (boolean), `fixed` (boolean, experimental)
- **Editor UI**: Position dropdown in sidebar
- **Note**: Sticky only works for root-level blocks
- **theme.json**: `settings.position.sticky`

#### renaming
- **Type**: `boolean`
- **Default**: `true`
- **Purpose**: Users can rename block instances in List View

#### reusable
- **Type**: `boolean`
- **Default**: `true`
- **Purpose**: Conversion to reusable block (Pattern)

#### splitting
- **Type**: `boolean`
- **Default**: `false`
- **Purpose**: Enter key splits block into two
- **Use when**: Simple text blocks with RichText

#### visibility
- **Type**: `boolean`
- **Default**: `true`
- **Purpose**: Block visibility controls

#### ariaLabel
- **Type**: `boolean`
- **Default**: `false`
- **Purpose**: Programmatic aria-label attribute
- **No UI**: Attribute only

## Supports vs Custom Controls Decision

Before building custom UI, check this list:

| You want to let users... | Support | Custom control needed? |
|---|---|---|
| Change text color | `color.text` | No |
| Change background color | `color.background` | No |
| Pick a gradient | `color.gradients` | No |
| Change font size | `typography.fontSize` | No |
| Change font family | `typography.fontFamily` | No |
| Adjust line height | `typography.lineHeight` | No |
| Align the block | `align` | No |
| Align text | `typography.textAlign` | No |
| Set margins | `spacing.margin` | No |
| Set padding | `spacing.padding` | No |
| Add a border | `border` | No |
| Add a shadow | `shadow` | No |
| Set minimum height | `dimensions.minHeight` | No |
| Set aspect ratio | `dimensions.aspectRatio` | No |
| Add anchor ID | `anchor` | No |
| Set background image | `background.backgroundImage` | No |
| Pick from custom options | — | Yes: SelectControl/RadioControl |
| Toggle a feature | — | Yes: ToggleControl |
| Enter free text | — | Yes: TextControl |
| Select data source | — | Yes: ComboboxControl/custom |
| Upload media | — | Yes: MediaPlaceholder/MediaUpload |

## Sources

- [Block Supports API](https://developer.wordpress.org/block-editor/reference-guides/block-api/block-supports/)
- [Block Supports in Static Blocks](https://developer.wordpress.org/block-editor/how-to-guides/block-tutorial/block-supports-in-static-blocks/)
- [WordPress Storybook](https://wordpress.github.io/gutenberg/)
