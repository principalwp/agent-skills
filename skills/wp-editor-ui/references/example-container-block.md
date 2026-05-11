---
wp_version_tested: 6.9
last_verified: 2026-04-04
---

# Gold Standard: Card Grid Block (container/layout archetype)

> Reference implementation demonstrating correct editor UI patterns for
> container blocks with inner blocks. Every comment explains **why**, not what.

## block.json

```json
{
  "$schema": "https://schemas.wp.org/trunk/block.json",
  "apiVersion": 3,
  "name": "my-plugin/card-grid",
  "version": "1.0.0",
  "title": "Card Grid",
  "category": "design",
  "icon": "grid-view",
  "description": "A responsive grid container for card blocks.",
  "textdomain": "my-plugin",
  "allowedBlocks": [ "my-plugin/card" ],
  "supports": {
    "anchor": true,
    "html": false,
    "align": [ "wide", "full" ],
    "color": {
      "background": true,
      "text": true,
      "link": true
    },
    "spacing": {
      "margin": true,
      "padding": true,
      "blockGap": true
    },
    "layout": {
      "default": {
        "type": "grid",
        "minimumColumnWidth": "16rem"
      },
      "allowSizingOnChildren": true
    },
    "dimensions": {
      "minHeight": true
    },
    "typography": {
      "fontSize": true,
      "lineHeight": true
    },
    "interactivity": {
      "clientNavigation": true
    }
  },
  "render": "file:./render.php",
  "editorScript": "file:./index.js",
  "editorStyle": "file:./index.css",
  "style": "file:./style-index.css"
}
```

### block.json decisions

| Decision | Rationale |
|----------|-----------|
| `layout` with `type: "grid"` | The grid layout type provides native column controls, responsive behavior, and `minimumColumnWidth` in the editor -- zero custom CSS grid code needed. |
| `minimumColumnWidth: "16rem"` | Auto-wrapping grid: cards never shrink below 16rem. The browser decides column count based on available width. No manual "columns" control needed. |
| `allowSizingOnChildren: true` | Lets editors set `columnSpan` on individual child cards to create featured/hero card layouts without custom controls. |
| `spacing.blockGap: true` | Gap between cards is a spacing concern. The supports system handles it with native UI in the Dimensions panel. No custom RangeControl needed. |
| `dimensions.minHeight: true` | Editors can ensure the grid has a minimum height for hero-style sections. Native support, no custom code. |
| `align: ["wide", "full"]` | Card grids are section-level elements that benefit from breaking out of content width. |
| `allowedBlocks` at block.json level | Declares which blocks can be inserted as children. This is enforced by the editor's inserter -- only `my-plugin/card` appears in the child block picker. |
| No custom attributes | Container blocks ideally have zero custom attributes. Everything is handled by supports: layout for grid, spacing for gap, dimensions for minHeight, color for theming. |
| `html: false` | Dynamic block with `render.php`. HTML editing mode is meaningless. |

## edit.js

```jsx
/**
 * Card Grid -- Edit component.
 *
 * Archetype: container block with inner blocks.
 * No custom sidebar controls -- supports provide everything.
 */
import { __ } from '@wordpress/i18n';
import {
	useBlockProps,
	useInnerBlocksProps,
} from '@wordpress/block-editor';
import { useSelect } from '@wordpress/data';
import { store as blockEditorStore } from '@wordpress/block-editor';

/**
 * Template for initial inner blocks. When a user inserts the Card Grid,
 * it starts with 3 cards so the grid layout is immediately visible.
 * Without a template, the empty state would show first -- fine, but less
 * useful for a container whose purpose is obvious.
 */
const INNER_BLOCKS_TEMPLATE = [
	[ 'my-plugin/card', {} ],
	[ 'my-plugin/card', {} ],
	[ 'my-plugin/card', {} ],
];

/**
 * Only allow card blocks as children. This is also declared in block.json
 * `allowedBlocks` for server-side enforcement, but must be repeated here
 * because useInnerBlocksProps reads from the JS config, not block.json,
 * when rendering the inserter UI.
 */
const ALLOWED_BLOCKS = [ 'my-plugin/card' ];

export default function Edit( { clientId } ) {
	// useBlockProps is REQUIRED on the outermost element. For container blocks
	// it merges with useInnerBlocksProps so supports (color, spacing, layout,
	// dimensions) all inject correctly into the single wrapper element.
	const blockProps = useBlockProps();

	// useInnerBlocksProps handles the inner blocks rendering area. Passing
	// blockProps as the first argument merges outer block attributes (from
	// supports) with inner block container attributes (from layout) onto a
	// single DOM element. This avoids a double-wrapper that would break
	// CSS grid layout.
	const innerBlocksProps = useInnerBlocksProps( blockProps, {
		allowedBlocks: ALLOWED_BLOCKS,
		template: INNER_BLOCKS_TEMPLATE,
		// "insert" orientation hint tells the editor to show the horizontal
		// insertion indicator between cards (matching the grid layout) rather
		// than the default vertical indicator.
		orientation: 'horizontal',
		// renderAppender is left at default (InnerBlocks.ButtonBlockAppender).
		// This renders the "+" button inside the grid, styled consistently
		// with core container blocks like Group and Columns.
	} );

	// Count child blocks to decide between empty state and populated grid.
	// useSelect is the correct way to read editor state -- never access
	// the block store imperatively.
	const childCount = useSelect(
		( select ) =>
			select( blockEditorStore ).getBlockCount( clientId ),
		[ clientId ]
	);

	// --- Empty state ---
	// When all cards have been removed, show a clear prompt. The inner blocks
	// props still render (providing the appender button) but we add context
	// so the editor understands what to do.
	//
	// We do NOT use <Placeholder> here because the block already has a valid
	// state (an empty grid is valid). Placeholder is for blocks that need
	// initial configuration before they can render anything. An empty container
	// just needs children added.
	if ( childCount === 0 ) {
		return (
			<div { ...innerBlocksProps }>
				{ /* The innerBlocksProps.children includes the appender.
				     We wrap it with instructional text. */ }
				<p className="wp-block-my-plugin-card-grid__empty-hint">
					{ __(
						'Add cards using the + button below.',
						'my-plugin'
					) }
				</p>
				{ innerBlocksProps.children }
			</div>
		);
	}

	// --- Populated grid ---
	// No custom rendering needed. The spread of innerBlocksProps provides:
	// - All supports-generated styles (from useBlockProps via the merge)
	// - Grid layout CSS (from the layout support)
	// - Block gap (from spacing.blockGap support)
	// - Min height (from dimensions.minHeight support)
	// - The inner blocks render area with child cards
	// - The appender button at the end
	//
	// This is the ideal container block: zero custom controls, zero custom
	// CSS for layout, everything from supports. The Edit component is pure
	// structure.
	return <div { ...innerBlocksProps } />;
}
```

## style.css (shared editor + frontend)

```css
/*
 * Minimal styles -- layout is handled entirely by the layout support.
 * The grid's column count, gap, and min-height are all injected by
 * WordPress via useBlockProps / get_block_wrapper_attributes.
 *
 * Only custom CSS needed is for the empty state hint and any decorative
 * concerns the supports don't cover.
 */

/* Empty state hint. Spans all grid columns so it centers properly
   regardless of the grid's column configuration. */
.wp-block-my-plugin-card-grid__empty-hint {
	grid-column: 1 / -1;
	text-align: center;
	padding: var(--wp--preset--spacing--40, 1.5rem);
	color: var(--wp--preset--color--contrast, currentColor);
	opacity: 0.5;
	font-size: var(--wp--preset--font-size--small, 0.875rem);
	margin: 0;
}
```

## render.php (server-side output)

```php
<?php
/**
 * Dynamic render callback for the Card Grid block.
 *
 * @param array    $attributes Block attributes.
 * @param string   $content    Inner block content (rendered child cards).
 * @param WP_Block $block      Block instance.
 */

// For container blocks, $content contains the already-rendered inner blocks.
// We just wrap it in the block's outer element with supports attributes.
//
// get_block_wrapper_attributes() is the PHP equivalent of useBlockProps().
// It outputs all supports-generated classes and inline styles: layout grid
// CSS, block gap, min height, colors, spacing, typography.
$wrapper_attributes = get_block_wrapper_attributes();

// Don't render an empty wrapper if there are no child blocks.
// trim() catches whitespace-only content from empty inner blocks.
if ( empty( trim( $content ) ) ) {
	return '';
}
?>
<div <?php echo $wrapper_attributes; ?>>
	<?php
	// $content is already escaped by the inner blocks' own render callbacks.
	// phpcs:ignore WordPress.Security.EscapeOutput.OutputNotEscaped
	echo $content;
	?>
</div>
```

## Companion: card child block (minimal)

The card grid requires `my-plugin/card` as its child block. Here is a minimal
block.json and edit.js for the child to make this example self-contained.

### card/block.json

```json
{
  "$schema": "https://schemas.wp.org/trunk/block.json",
  "apiVersion": 3,
  "name": "my-plugin/card",
  "version": "1.0.0",
  "title": "Card",
  "category": "design",
  "icon": "cover-image",
  "description": "A single card within a Card Grid.",
  "textdomain": "my-plugin",
  "parent": [ "my-plugin/card-grid" ],
  "attributes": {
    "heading": {
      "type": "string",
      "source": "rich-text",
      "selector": ".wp-block-my-plugin-card__heading"
    },
    "content": {
      "type": "string",
      "source": "rich-text",
      "selector": ".wp-block-my-plugin-card__content"
    }
  },
  "supports": {
    "anchor": true,
    "html": false,
    "color": {
      "background": true,
      "text": true,
      "link": true,
      "gradients": true
    },
    "spacing": {
      "padding": true
    },
    "border": {
      "color": true,
      "radius": true,
      "width": true
    },
    "shadow": true,
    "interactivity": {
      "clientNavigation": true
    }
  },
  "render": "file:./render.php",
  "editorScript": "file:./index.js",
  "style": "file:./style-index.css"
}
```

### card/edit.js

```jsx
/**
 * Card -- Edit component (child of Card Grid).
 *
 * Uses RichText for direct content manipulation. No sidebar controls --
 * all card styling comes from supports (color, border, shadow, padding).
 */
import { __ } from '@wordpress/i18n';
import { useBlockProps, RichText } from '@wordpress/block-editor';

export default function Edit( { attributes, setAttributes } ) {
	const blockProps = useBlockProps();

	return (
		<div { ...blockProps }>
			<RichText
				tagName="h3"
				className="wp-block-my-plugin-card__heading"
				value={ attributes.heading }
				onChange={ ( value ) =>
					setAttributes( { heading: value } )
				}
				placeholder={ __( 'Card title...', 'my-plugin' ) }
				allowedFormats={ [ 'core/bold', 'core/italic' ] }
			/>
			<RichText
				tagName="p"
				className="wp-block-my-plugin-card__content"
				value={ attributes.content }
				onChange={ ( value ) =>
					setAttributes( { content: value } )
				}
				placeholder={ __( 'Card content...', 'my-plugin' ) }
			/>
		</div>
	);
}
```

## Key Decisions

| Decision | Rationale |
|----------|-----------|
| No custom sidebar controls | Container blocks should delegate all styling to supports. Layout, gap, min-height, colors -- the native UI handles all of these with better consistency than custom controls. |
| `layout` support with `type: "grid"` | Provides responsive grid behavior with `minimumColumnWidth` auto-wrapping. No manual "columns" RangeControl needed; the browser adapts automatically. |
| `useInnerBlocksProps( blockProps, ... )` merged call | Single wrapper element serves both as the block container and the inner blocks area. A double-wrapper (separate `blockProps` div + `innerBlocksProps` div) would break CSS grid layout because the grid properties would be on the outer div while children are in the inner div. |
| `allowedBlocks` in both block.json and JS | block.json `allowedBlocks` is the server-side source of truth (new in WP 6.9). JS `allowedBlocks` passed to `useInnerBlocksProps` is needed because the editor inserter reads from the JS config for the block picker UI. Both must agree. |
| `parent: ["my-plugin/card-grid"]` on child | Prevents the Card block from appearing in the inserter outside of a Card Grid. Enforced by the editor -- users can only add cards inside grids. |
| `orientation: "horizontal"` | Visual hint for the editor's drag-and-drop insertion indicator. Horizontal shows the vertical blue line between grid cells instead of a horizontal line between rows. |
| No `<Placeholder>` for empty state | Placeholder signals "needs configuration before rendering." An empty grid is already a valid state -- it just needs children. Instructional text + the default appender is the correct pattern. |
| `INNER_BLOCKS_TEMPLATE` with 3 cards | Immediate visual feedback when inserting. An empty grid with just an appender button is less clear than 3 starter cards that demonstrate the layout. |
| `childCount === 0` check via `useSelect` | `useSelect` reads from the block editor store. Never access store state imperatively or count `innerBlocksProps.children` -- that includes the appender and doesn't reflect actual block count. |
| Trim check in render.php | Empty inner blocks may produce whitespace-only `$content`. Rendering an empty wrapper div creates invisible layout artifacts (margins, padding from supports). |
| `allowSizingOnChildren: true` | Unlocks `columnSpan` controls on child cards. Editors can make one card span 2 columns for a featured layout without any custom code in the parent or child. |
| Card child uses `parent` constraint | `parent` is the inverse of `allowedBlocks`. Together they create a bidirectional lock: the grid only accepts cards, and cards only appear inside grids. Belt-and-suspenders enforcement. |
