---
wp_version_tested: 6.9
last_verified: 2026-04-04
---

# Gold Standard: Latest Testimonials Block (data-fetching archetype)

> Reference implementation demonstrating correct editor UI patterns for
> data-fetching blocks. Every comment explains **why**, not what.

## block.json

```json
{
  "$schema": "https://schemas.wp.org/trunk/block.json",
  "apiVersion": 3,
  "name": "my-plugin/latest-testimonials",
  "version": "1.0.0",
  "title": "Latest Testimonials",
  "category": "widgets",
  "icon": "format-quote",
  "description": "Display recent testimonials from a selected source.",
  "textdomain": "my-plugin",
  "attributes": {
    "postType": {
      "type": "string",
      "default": ""
    },
    "maxItems": {
      "type": "number"
    },
    "showAuthor": {
      "type": "boolean"
    },
    "showDate": {
      "type": "boolean"
    },
    "columns": {
      "type": "number"
    }
  },
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
| `"html": false` | Dynamic block -- HTML editing mode is meaningless when `save()` returns null. |
| `anchor: true` | Low-cost deep linking. Always include unless the block is never a navigation target. |
| No `align` support | Testimonials render within content flow; wide/full adds layout complexity without clear benefit for a data list. |
| `interactivity.clientNavigation` | Declares compatibility with client-side navigation. Costs nothing; future-proofs. |
| Attributes without defaults for optional settings | `undefined` means "not customized" -- `ToolsPanelItem.hasValue` returns false, keeping the sidebar clean via progressive disclosure. |
| `"render": "file:./render.php"` | Dynamic rendering. Design changes propagate to every instance without re-saving posts. |
| Color + typography + spacing only | Matches supports-catalog recommendation for "data display" blocks. Borders and shadows are not typical for feed-style output. |

## edit.js

```jsx
/**
 * Latest Testimonials -- Edit component.
 *
 * Archetype: data-fetching block with all four states.
 */
import { __ } from '@wordpress/i18n';
import { useBlockProps, InspectorControls } from '@wordpress/block-editor';
import {
	ComboboxControl,
	Notice,
	Placeholder,
	RangeControl,
	Spinner,
	ToggleControl,
	// ToolsPanel is stable API used by all core blocks, but still exported
	// under __experimental prefix. Import with alias per convention.
	__experimentalToolsPanel as ToolsPanel,
	__experimentalToolsPanelItem as ToolsPanelItem,
} from '@wordpress/components';
import { useEntityRecords } from '@wordpress/core-data';
import { useCallback, useMemo } from '@wordpress/element';

/**
 * Post types eligible for testimonial display. In a real plugin this would
 * come from a REST discovery call or a server-side filter; hardcoded here
 * to keep the example focused on editor UI patterns.
 */
const POST_TYPE_OPTIONS = [
	{ value: 'testimonial', label: __( 'Testimonials', 'my-plugin' ) },
	{ value: 'review', label: __( 'Reviews', 'my-plugin' ) },
	{ value: 'case-study', label: __( 'Case Studies', 'my-plugin' ) },
];

/** Attribute defaults live here, not in block.json, so ToolsPanel can
 *  distinguish "explicitly set" from "inherited default." When an attribute
 *  is `undefined`, ToolsPanelItem.hasValue returns false and the control
 *  hides behind the panel menu -- progressive disclosure. */
const DEFAULTS = {
	maxItems: 3,
	showAuthor: true,
	showDate: false,
	columns: 1,
};

export default function Edit( { attributes, setAttributes } ) {
	// useBlockProps is REQUIRED on the outermost element of EVERY render path.
	// All supports (color, spacing, typography) inject their styles through it.
	const blockProps = useBlockProps();

	const { postType, maxItems, showAuthor, showDate, columns } = attributes;

	// Resolve display values: attribute if set, otherwise default.
	const displayMaxItems = maxItems ?? DEFAULTS.maxItems;
	const displayShowAuthor = showAuthor ?? DEFAULTS.showAuthor;
	const displayShowDate = showDate ?? DEFAULTS.showDate;
	const displayColumns = columns ?? DEFAULTS.columns;

	// --- Data fetching via useEntityRecords ---
	// Preferred over raw apiFetch because it integrates with the editor's
	// entity cache, deduplicates requests, and auto-updates on post saves.
	const query = useMemo(
		() => ( {
			per_page: displayMaxItems,
			orderby: 'date',
			order: 'desc',
			_embed: true, // Sideloads author and featured media in one request.
		} ),
		[ displayMaxItems ]
	);

	const { records, isResolving, hasResolved } = useEntityRecords(
		'postType',
		postType,
		// Only fetch when postType is selected. Passing null skips the request.
		postType ? query : null
	);

	const isLoading = postType && isResolving;
	const hasError = postType && hasResolved && records === null;
	const hasNoPosts = postType && hasResolved && records?.length === 0;

	// --- ToolsPanel reset callback ---
	// Wrapped in useCallback because it's passed as a prop -- avoids
	// unnecessary child re-renders. The performance reviewer will flag
	// inline arrow functions passed as props.
	const resetAll = useCallback( () => {
		setAttributes( {
			maxItems: undefined,
			showAuthor: undefined,
			showDate: undefined,
			columns: undefined,
		} );
	}, [ setAttributes ] );

	// --- State 1: Placeholder (no data source selected) ---
	// Grey background signals "needs configuration." ComboboxControl is
	// correct here because the options list could grow large (many CPTs);
	// SelectControl would work for 3-6 options but ComboboxControl adds
	// search for free and scales better.
	if ( ! postType ) {
		return (
			<div { ...blockProps }>
				<Placeholder
					icon="format-quote"
					label={ __( 'Latest Testimonials', 'my-plugin' ) }
					instructions={ __(
						'Choose which post type to display testimonials from.',
						'my-plugin'
					) }
				>
					<ComboboxControl
						__next40pxDefaultSize
						__nextHasNoMarginBottom
						label={ __( 'Post type', 'my-plugin' ) }
						options={ POST_TYPE_OPTIONS }
						value={ postType }
						onChange={ ( value ) =>
							setAttributes( { postType: value } )
						}
					/>
				</Placeholder>
			</div>
		);
	}

	// --- State 2: Loading ---
	// Spinner inside Placeholder is the standard core pattern for brief,
	// unpredictable-layout loading. For blocks with a known layout shape,
	// use structural skeleton placeholders instead (see polish-patterns.md).
	if ( isLoading ) {
		return (
			<div { ...blockProps }>
				<Placeholder
					icon="format-quote"
					label={ __( 'Latest Testimonials', 'my-plugin' ) }
				>
					<Spinner />
				</Placeholder>
			</div>
		);
	}

	// --- State 3: Error ---
	// Never silently swallow errors. Every failed fetch must render
	// user-visible feedback. isDismissible=false because dismissing
	// would leave an empty block with no way to recover.
	if ( hasError ) {
		return (
			<div { ...blockProps }>
				<Notice status="error" isDismissible={ false }>
					{ __(
						'Testimonials could not be loaded. Verify the post type exists and has published posts.',
						'my-plugin'
					) }
				</Notice>
			</div>
		);
	}

	// --- State 4: Live preview ---
	// The preview should match the frontend render as closely as possible.
	// Sidebar controls use ToolsPanel (not PanelBody) because we have 4
	// optional settings -- progressive disclosure keeps the sidebar clean.
	return (
		<div { ...blockProps }>
			{ /* --- Sidebar: ToolsPanel for 3+ optional settings --- */ }
			<InspectorControls>
				<ToolsPanel
					label={ __( 'Display settings', 'my-plugin' ) }
					resetAll={ resetAll }
				>
					{ /* Max items: RangeControl gives visual feedback for
					     bounded numeric values. TextControl type="number"
					     would work but lacks the slider affordance. */ }
					<ToolsPanelItem
						label={ __( 'Max items', 'my-plugin' ) }
						hasValue={ () => maxItems !== undefined }
						onDeselect={ () =>
							setAttributes( { maxItems: undefined } )
						}
						isShownByDefault
					>
						<RangeControl
							__next40pxDefaultSize
							__nextHasNoMarginBottom
							label={ __( 'Max items', 'my-plugin' ) }
							value={ displayMaxItems }
							onChange={ ( value ) =>
								setAttributes( { maxItems: value } )
							}
							min={ 1 }
							max={ 12 }
						/>
					</ToolsPanelItem>

					{ /* Show author: ToggleControl for booleans.
					     __nextHasNoMarginBottom removes legacy spacing. */ }
					<ToolsPanelItem
						label={ __( 'Show author', 'my-plugin' ) }
						hasValue={ () => showAuthor !== undefined }
						onDeselect={ () =>
							setAttributes( { showAuthor: undefined } )
						}
						isShownByDefault
					>
						<ToggleControl
							__nextHasNoMarginBottom
							label={ __( 'Show author', 'my-plugin' ) }
							checked={ displayShowAuthor }
							onChange={ ( value ) =>
								setAttributes( { showAuthor: value } )
							}
						/>
					</ToolsPanelItem>

					{ /* Show date: Not shown by default -- tertiary option.
					     Appears in ToolsPanel dropdown menu until activated. */ }
					<ToolsPanelItem
						label={ __( 'Show date', 'my-plugin' ) }
						hasValue={ () => showDate !== undefined }
						onDeselect={ () =>
							setAttributes( { showDate: undefined } )
						}
					>
						<ToggleControl
							__nextHasNoMarginBottom
							label={ __( 'Show date', 'my-plugin' ) }
							checked={ displayShowDate }
							onChange={ ( value ) =>
								setAttributes( { showDate: value } )
							}
						/>
					</ToolsPanelItem>

					{ /* Columns: Only relevant when maxItems > 1. Conditional
					     rendering avoids showing a useless control. */ }
					{ displayMaxItems > 1 && (
						<ToolsPanelItem
							label={ __( 'Columns', 'my-plugin' ) }
							hasValue={ () => columns !== undefined }
							onDeselect={ () =>
								setAttributes( { columns: undefined } )
							}
						>
							<RangeControl
								__next40pxDefaultSize
								__nextHasNoMarginBottom
								label={ __( 'Columns', 'my-plugin' ) }
								value={ displayColumns }
								onChange={ ( value ) =>
									setAttributes( { columns: value } )
								}
								min={ 1 }
								max={ 4 }
							/>
						</ToolsPanelItem>
					) }
				</ToolsPanel>
			</InspectorControls>

			{ /* --- Canvas: live preview of fetched data --- */ }
			{ hasNoPosts ? (
				<p>
					{ __(
						'No testimonials found. Publish some posts to see them here.',
						'my-plugin'
					) }
				</p>
			) : (
				<div
					className="wp-block-my-plugin-latest-testimonials__grid"
					style={ {
						// CSS custom properties with fallbacks -- adapts to
						// any theme. Never hardcode pixel values for spacing
						// or colors.
						'--testimonials-columns': displayColumns,
						'--testimonials-gap':
							'var(--wp--preset--spacing--30, 1rem)',
					} }
				>
					{ records?.map( ( post ) => (
						<article
							key={ post.id }
							className="wp-block-my-plugin-latest-testimonials__item"
						>
							{ /* dangerouslySetInnerHTML is safe here:
							     content comes from the WP REST API which
							     already runs wp_kses on output. */ }
							<blockquote
								className="wp-block-my-plugin-latest-testimonials__quote"
								dangerouslySetInnerHTML={ {
									__html: post.content?.rendered,
								} }
							/>
							{ displayShowAuthor && (
								<cite className="wp-block-my-plugin-latest-testimonials__author">
									{ post._embedded?.author?.[ 0 ]
										?.name ??
										__( 'Anonymous', 'my-plugin' ) }
								</cite>
							) }
							{ displayShowDate && (
								<time
									className="wp-block-my-plugin-latest-testimonials__date"
									dateTime={ post.date }
								>
									{ new Date(
										post.date
									).toLocaleDateString() }
								</time>
							) }
						</article>
					) ) }
				</div>
			) }
		</div>
	);
}
```

## style.css (shared editor + frontend)

```css
/*
 * CSS custom properties with fallbacks -- adapts to any theme.
 * Never hardcode hex colors; always reference preset or custom vars.
 */
.wp-block-my-plugin-latest-testimonials__grid {
	display: grid;
	grid-template-columns: repeat(
		var(--testimonials-columns, 1),
		1fr
	);
	gap: var(--testimonials-gap, var(--wp--preset--spacing--30, 1rem));
}

.wp-block-my-plugin-latest-testimonials__item {
	display: flex;
	flex-direction: column;
	gap: var(--wp--preset--spacing--20, 0.5rem);
}

.wp-block-my-plugin-latest-testimonials__quote {
	margin: 0;
	font-size: var(--wp--preset--font-size--medium, 1rem);
	color: var(--wp--preset--color--contrast, currentColor);
}

.wp-block-my-plugin-latest-testimonials__author {
	font-style: normal;
	font-weight: 600;
	font-size: var(--wp--preset--font-size--small, 0.875rem);
}

.wp-block-my-plugin-latest-testimonials__date {
	font-size: var(--wp--preset--font-size--small, 0.875rem);
	color: var(--wp--preset--color--contrast, currentColor);
	opacity: 0.6;
}
```

## render.php (server-side output)

```php
<?php
/**
 * Dynamic render callback for the Latest Testimonials block.
 *
 * @param array    $attributes Block attributes.
 * @param string   $content    Inner content (empty for this block).
 * @param WP_Block $block      Block instance.
 */

// Defaults match the JS DEFAULTS object -- keep them in sync.
$max_items   = $attributes['maxItems'] ?? 3;
$show_author = $attributes['showAuthor'] ?? true;
$show_date   = $attributes['showDate'] ?? false;
$columns     = $attributes['columns'] ?? 1;
$post_type   = $attributes['postType'] ?? '';

if ( empty( $post_type ) || ! post_type_exists( $post_type ) ) {
	return '';
}

$query = new WP_Query( [
	'post_type'      => $post_type,
	'posts_per_page' => $max_items,
	'orderby'        => 'date',
	'order'          => 'DESC',
	'no_found_rows'  => true, // Performance: skip pagination count.
] );

if ( ! $query->have_posts() ) {
	return '';
}

// get_block_wrapper_attributes() is the PHP equivalent of useBlockProps().
// All supports (color, spacing, typography) inject their styles through it.
$wrapper_attributes = get_block_wrapper_attributes();

$style = sprintf(
	'--testimonials-columns:%d;--testimonials-gap:var(--wp--preset--spacing--30, 1rem);',
	$columns
);

ob_start();
?>
<div <?php echo $wrapper_attributes; ?>>
	<div class="wp-block-my-plugin-latest-testimonials__grid" style="<?php echo esc_attr( $style ); ?>">
		<?php while ( $query->have_posts() ) : $query->the_post(); ?>
			<article class="wp-block-my-plugin-latest-testimonials__item">
				<blockquote class="wp-block-my-plugin-latest-testimonials__quote">
					<?php the_content(); ?>
				</blockquote>
				<?php if ( $show_author ) : ?>
					<cite class="wp-block-my-plugin-latest-testimonials__author">
						<?php echo esc_html( get_the_author() ); ?>
					</cite>
				<?php endif; ?>
				<?php if ( $show_date ) : ?>
					<time class="wp-block-my-plugin-latest-testimonials__date" datetime="<?php echo esc_attr( get_the_date( 'c' ) ); ?>">
						<?php echo esc_html( get_the_date() ); ?>
					</time>
				<?php endif; ?>
			</article>
		<?php endwhile; ?>
	</div>
</div>
<?php
wp_reset_postdata();
echo ob_get_clean();
```

## Key Decisions

| Decision | Rationale |
|----------|-----------|
| ToolsPanel not PanelBody | 4 optional settings -- progressive disclosure keeps sidebar clean. PanelBody would show all controls at once. |
| ComboboxControl in Placeholder | Data source selection must happen before any preview. ComboboxControl scales to many post types with search; SelectControl caps at ~6 before UX degrades. |
| `useEntityRecords` not `apiFetch` | Integrates with editor entity cache, deduplicates requests, auto-updates on post saves. Raw `apiFetch` bypasses all of this. |
| `undefined` attribute defaults | Enables ToolsPanelItem `hasValue` to distinguish "user set this" from "using default." If defaults were in `block.json`, every attribute would appear "customized." |
| `isShownByDefault` on maxItems and showAuthor only | Most-used controls visible immediately; showDate and columns are tertiary options hidden behind the panel menu. |
| Conditional columns control | Showing a "Columns" slider when maxItems is 1 is nonsensical. Conditional rendering avoids user confusion. |
| `dangerouslySetInnerHTML` for post content | REST API content is already sanitized by `wp_kses`. Re-sanitizing in JS would strip intentional HTML. This is the same pattern `core/latest-posts` uses. |
| Spinner not skeleton loading | Testimonial count and layout vary per configuration -- no predictable shape to skeleton. Spinner + Placeholder is the correct pattern for unpredictable layouts. |
| `no_found_rows` in PHP query | Performance optimization: skips the `SQL_CALC_FOUND_ROWS` count since this block never paginates. Important on high-traffic sites at scale. |
| CSS custom properties with fallbacks | Adapts to any theme automatically. Hardcoded values create maintenance debt and break with theme switches. |
| `save()` returns null (omitted) | Dynamic block: all rendering happens server-side in `render.php`. Design updates propagate without re-saving posts. |
