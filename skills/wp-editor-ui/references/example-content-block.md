---
wp_version_tested: 6.9
last_verified: 2026-04-04
---

# Gold Standard: Call to Action Block (content-editing archetype)

> Reference implementation demonstrating correct editor UI patterns for
> content-editing blocks. Every comment explains **why**, not what.

## block.json

```json
{
  "$schema": "https://schemas.wp.org/trunk/block.json",
  "apiVersion": 3,
  "name": "my-plugin/call-to-action",
  "version": "1.0.0",
  "title": "Call to Action",
  "category": "design",
  "icon": "megaphone",
  "description": "A prominent section with heading, description, and action button.",
  "textdomain": "my-plugin",
  "attributes": {
    "heading": {
      "type": "string",
      "source": "rich-text",
      "selector": ".wp-block-my-plugin-call-to-action__heading"
    },
    "description": {
      "type": "string",
      "source": "rich-text",
      "selector": ".wp-block-my-plugin-call-to-action__description"
    },
    "buttonText": {
      "type": "string",
      "source": "rich-text",
      "selector": ".wp-block-my-plugin-call-to-action__button"
    },
    "buttonUrl": {
      "type": "string",
      "default": ""
    },
    "openInNewTab": {
      "type": "boolean",
      "default": false
    },
    "mediaId": {
      "type": "number"
    },
    "mediaUrl": {
      "type": "string"
    },
    "mediaAlt": {
      "type": "string",
      "default": ""
    },
    "textAlign": {
      "type": "string"
    }
  },
  "supports": {
    "anchor": true,
    "html": false,
    "align": [ "wide", "full" ],
    "color": {
      "background": true,
      "text": true,
      "link": true,
      "gradients": true
    },
    "spacing": {
      "margin": true,
      "padding": true
    },
    "typography": {
      "fontSize": true,
      "lineHeight": true,
      "fontFamily": true
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
  "editorStyle": "file:./index.css",
  "style": "file:./style-index.css"
}
```

### block.json decisions

| Decision | Rationale |
|----------|-----------|
| `align: ["wide", "full"]` | CTAs are hero-style elements that benefit from breaking out of content width. Restricting to wide/full prevents awkward left/center/right alignment on a full-width section. |
| `color.gradients: true` | CTAs commonly use gradient backgrounds for visual impact. The support provides a native gradient picker at zero custom code cost. |
| `typography.fontFamily` | CTAs often use display typefaces distinct from body text. FontFamily support lets editors pick from theme.json-registered fonts. |
| `border` + `shadow` | Decorative supports for a design-focused block. The native UI is better than custom controls for these common needs. |
| `source: "rich-text"` on text attributes | Enables persistence of inline formatting (bold, italic, links) through the block's RichText fields. Without this, formatting is lost on save. |
| `mediaId` + `mediaUrl` + `mediaAlt` | Standard media attribute trio: `mediaId` for library reference, `mediaUrl` for rendering, `mediaAlt` for accessibility. All three are needed for proper media handling. |
| No `textAlign` in supports | We manage text alignment via BlockControls toolbar toggle to keep it in the toolbar tier (priority 2) rather than buried in the sidebar. Supports-based `typography.textAlign` would place it in the sidebar typography panel. |

## edit.js

```jsx
/**
 * Call to Action -- Edit component.
 *
 * Archetype: content-editing block with direct canvas manipulation.
 * Most interaction happens inline; sidebar is minimal.
 */
import { __ } from '@wordpress/i18n';
import {
	useBlockProps,
	BlockControls,
	InspectorControls,
	RichText,
	MediaPlaceholder,
	MediaUpload,
	MediaUploadCheck,
} from '@wordpress/block-editor';
import {
	PanelBody,
	TextControl,
	ToggleControl,
	ToolbarGroup,
	ToolbarButton,
} from '@wordpress/components';
import {
	alignLeft,
	alignCenter,
	alignRight,
	image as imageIcon,
} from '@wordpress/icons';

/** Alignment options for the toolbar toggle. */
const ALIGNMENT_CONTROLS = [
	{
		icon: alignLeft,
		title: __( 'Align text left', 'my-plugin' ),
		align: 'left',
	},
	{
		icon: alignCenter,
		title: __( 'Align text center', 'my-plugin' ),
		align: 'center',
	},
	{
		icon: alignRight,
		title: __( 'Align text right', 'my-plugin' ),
		align: 'right',
	},
];

export default function Edit( { attributes, setAttributes } ) {
	// useBlockProps MUST appear on the outermost element of EVERY render path.
	// All supports (color, spacing, typography, border, shadow) inject their
	// styles through it. Forgetting this silently breaks all supports.
	const blockProps = useBlockProps( {
		// className and style are merged into blockProps, not replaced.
		className: attributes.textAlign
			? `has-text-align-${ attributes.textAlign }`
			: undefined,
	} );

	const {
		heading,
		description,
		buttonText,
		buttonUrl,
		openInNewTab,
		mediaId,
		mediaUrl,
		mediaAlt,
		textAlign,
	} = attributes;

	const hasMedia = !! mediaUrl;

	const onSelectMedia = ( media ) => {
		// Guard against media library returning unexpected shapes.
		if ( ! media || ! media.url ) {
			return;
		}
		setAttributes( {
			mediaId: media.id,
			mediaUrl: media.url,
			// Use the library's alt text as default; editors can override.
			mediaAlt: media.alt || '',
		} );
	};

	const onRemoveMedia = () => {
		setAttributes( {
			mediaId: undefined,
			mediaUrl: undefined,
			mediaAlt: '',
		} );
	};

	return (
		<div { ...blockProps }>
			{ /* --- Toolbar: alignment toggle + media replace ---
			     Alignment lives in the toolbar (tier 2) because it's a
			     frequent structural action. Putting it in the sidebar would
			     violate the three-tier hierarchy: if a control is needed for
			     basic usage, it MUST be in canvas or toolbar. */ }
			<BlockControls>
				<ToolbarGroup>
					{ ALIGNMENT_CONTROLS.map( ( control ) => (
						<ToolbarButton
							key={ control.align }
							icon={ control.icon }
							label={ control.title }
							isActive={ textAlign === control.align }
							onClick={ () =>
								setAttributes( {
									textAlign:
										textAlign === control.align
											? undefined
											: control.align,
								} )
							}
						/>
					) ) }
				</ToolbarGroup>

				{ /* Media replace button: only shown when an image exists.
				     MediaUploadCheck wraps the trigger to hide it from users
				     without upload_files capability. */ }
				{ hasMedia && (
					<ToolbarGroup>
						<MediaUploadCheck>
							<MediaUpload
								onSelect={ onSelectMedia }
								allowedTypes={ [ 'image' ] }
								value={ mediaId }
								render={ ( { open } ) => (
									<ToolbarButton
										icon={ imageIcon }
										label={ __(
											'Replace image',
											'my-plugin'
										) }
										onClick={ open }
									/>
								) }
							/>
						</MediaUploadCheck>
						<ToolbarButton
							label={ __( 'Remove image', 'my-plugin' ) }
							onClick={ onRemoveMedia }
							isDestructive
						>
							{ __( 'Remove', 'my-plugin' ) }
						</ToolbarButton>
					</ToolbarGroup>
				) }
			</BlockControls>

			{ /* --- Sidebar: minimal, 1-2 settings ---
			     PanelBody (not ToolsPanel) is correct here because we have
			     only 2 required settings. ToolsPanel's progressive disclosure
			     adds complexity without benefit for small control sets. */ }
			<InspectorControls>
				<PanelBody
					title={ __( 'Link settings', 'my-plugin' ) }
					initialOpen
				>
					{ /* TextControl with __next40pxDefaultSize for modern
					     40px height. label prop is required -- never rely
					     on placeholder as the accessible label. */ }
					<TextControl
						__next40pxDefaultSize
						__nextHasNoMarginBottom
						label={ __( 'Button URL', 'my-plugin' ) }
						value={ buttonUrl }
						onChange={ ( value ) =>
							setAttributes( { buttonUrl: value } )
						}
						type="url"
						help={ __(
							'Where the button links to.',
							'my-plugin'
						) }
					/>
					<ToggleControl
						__nextHasNoMarginBottom
						label={ __( 'Open in new tab', 'my-plugin' ) }
						checked={ openInNewTab }
						onChange={ ( value ) =>
							setAttributes( { openInNewTab: value } )
						}
					/>
				</PanelBody>
			</InspectorControls>

			{ /* --- Canvas: direct manipulation (tier 1) ---
			     Content editing happens inline on the canvas. This is the
			     primary interface -- no sidebar trip required to edit text.
			     The three-tier hierarchy says: if a control is required for
			     basic block usage, it MUST be in the content area. */ }

			{ /* Background image area. MediaPlaceholder provides drag-drop,
			     media library, and URL input. It only renders when no image
			     is selected -- once selected, the image renders as a
			     background. */ }
			{ ! hasMedia ? (
				<MediaPlaceholder
					icon={ imageIcon }
					labels={ {
						title: __( 'Background image', 'my-plugin' ),
						instructions: __(
							'Drag an image, upload, or select from the media library. This is optional.',
							'my-plugin'
						),
					} }
					onSelect={ onSelectMedia }
					accept="image/*"
					allowedTypes={ [ 'image' ] }
					// disableMediaButtons would remove the upload button;
					// we want all three input methods available.
				/>
			) : (
				<div
					className="wp-block-my-plugin-call-to-action__media"
					style={ {
						backgroundImage: `url(${ mediaUrl })`,
					} }
					role="img"
					aria-label={ mediaAlt }
				/>
			) }

			<div className="wp-block-my-plugin-call-to-action__content">
				{ /* RichText for heading: direct manipulation on canvas.
				     tagName="h2" renders semantically correct heading.
				     allowedFormats restricts to bold/italic -- headings
				     should not contain links or code. */ }
				<RichText
					tagName="h2"
					className="wp-block-my-plugin-call-to-action__heading"
					value={ heading }
					onChange={ ( value ) =>
						setAttributes( { heading: value } )
					}
					placeholder={ __(
						'Write a compelling headline...',
						'my-plugin'
					) }
					allowedFormats={ [ 'core/bold', 'core/italic' ] }
				/>

				{ /* RichText for description: full formatting allowed
				     since body text legitimately needs links and inline
				     code. */ }
				<RichText
					tagName="p"
					className="wp-block-my-plugin-call-to-action__description"
					value={ description }
					onChange={ ( value ) =>
						setAttributes( { description: value } )
					}
					placeholder={ __(
						'Add a supporting description...',
						'my-plugin'
					) }
				/>

				{ /* RichText for button text: inline editing avoids a
				     sidebar trip for the most common action. The button
				     is not a real link in the editor -- clicking it would
				     navigate away. Rendering as a <span> in edit, <a> in
				     render.php is the standard pattern. */ }
				<RichText
					tagName="span"
					className="wp-block-my-plugin-call-to-action__button"
					value={ buttonText }
					onChange={ ( value ) =>
						setAttributes( { buttonText: value } )
					}
					placeholder={ __( 'Button text...', 'my-plugin' ) }
					allowedFormats={ [] }
				/>
			</div>
		</div>
	);
}
```

## style.css (shared editor + frontend)

```css
/*
 * All values reference CSS custom properties with fallbacks.
 * Never hardcode hex colors -- always use preset or custom vars.
 */
.wp-block-my-plugin-call-to-action {
	position: relative;
	display: flex;
	flex-direction: column;
	align-items: stretch;
	overflow: hidden;
	border-radius: var(--wp--custom--border-radius--default, 0.5rem);
}

/* Background image layer. Uses object-like positioning via background
   properties so the image fills the block regardless of aspect ratio. */
.wp-block-my-plugin-call-to-action__media {
	position: absolute;
	inset: 0;
	background-size: cover;
	background-position: center;
	z-index: 0;
}

/* Semi-transparent overlay over the image so text remains readable.
   color-mix() is the modern Gutenberg pattern (see polish-patterns.md). */
.wp-block-my-plugin-call-to-action__media::after {
	content: "";
	position: absolute;
	inset: 0;
	background: color-mix(
		in srgb,
		var(--wp--preset--color--contrast, #000) 50%,
		transparent
	);
}

.wp-block-my-plugin-call-to-action__content {
	position: relative;
	z-index: 1;
	display: flex;
	flex-direction: column;
	gap: var(--wp--preset--spacing--20, 0.5rem);
	padding: var(--wp--preset--spacing--50, 2rem);
}

.wp-block-my-plugin-call-to-action__heading {
	margin: 0;
	font-size: var(--wp--preset--font-size--x-large, 2rem);
	color: var(--wp--preset--color--base, #fff);
}

.wp-block-my-plugin-call-to-action__description {
	margin: 0;
	font-size: var(--wp--preset--font-size--medium, 1rem);
	color: var(--wp--preset--color--base, #fff);
	opacity: 0.9;
}

/* Button element. Uses the theme's accent color so it adapts automatically.
   Transition follows Gutenberg's button timing (100ms for box-shadow,
   50ms for background). Reduced motion users get no transition. */
.wp-block-my-plugin-call-to-action__button {
	display: inline-block;
	align-self: flex-start;
	padding: var(--wp--preset--spacing--20, 0.5rem) var(--wp--preset--spacing--40, 1.5rem);
	background: var(--wp--preset--color--primary, var(--wp-admin-theme-color, #3858e9));
	color: var(--wp--preset--color--base, #fff);
	font-size: var(--wp--preset--font-size--small, 0.875rem);
	font-weight: 600;
	text-decoration: none;
	border-radius: var(--wp--custom--border-radius--small, 0.25rem);
	cursor: pointer;
}

@media not (prefers-reduced-motion) {
	.wp-block-my-plugin-call-to-action__button {
		transition:
			box-shadow 100ms linear,
			background 50ms ease-in-out;
	}
}

/* Hover uses color-mix pattern from polish-patterns.md.
   Only applied to the frontend anchor element, not the editor span. */
a.wp-block-my-plugin-call-to-action__button:hover {
	background: color-mix(
		in srgb,
		var(--wp--preset--color--primary, #3858e9) 85%,
		#000
	);
}

a.wp-block-my-plugin-call-to-action__button:focus-visible {
	box-shadow: 0 0 0 2px var(--wp--preset--color--base, #fff),
	            0 0 0 4px var(--wp--preset--color--primary, #3858e9);
	outline: 2px solid transparent;
}

/* Text alignment classes. Applied by the toolbar toggle. */
.wp-block-my-plugin-call-to-action.has-text-align-center .wp-block-my-plugin-call-to-action__content {
	align-items: center;
	text-align: center;
}

.wp-block-my-plugin-call-to-action.has-text-align-right .wp-block-my-plugin-call-to-action__content {
	align-items: flex-end;
	text-align: right;
}
```

## render.php (server-side output)

```php
<?php
/**
 * Dynamic render callback for the Call to Action block.
 *
 * @param array    $attributes Block attributes.
 * @param string   $content    Inner content (empty for this block).
 * @param WP_Block $block      Block instance.
 */

$heading      = $attributes['heading'] ?? '';
$description  = $attributes['description'] ?? '';
$button_text  = $attributes['buttonText'] ?? '';
$button_url   = $attributes['buttonUrl'] ?? '';
$open_new_tab = $attributes['openInNewTab'] ?? false;
$media_url    = $attributes['mediaUrl'] ?? '';
$media_alt    = $attributes['mediaAlt'] ?? '';
$text_align   = $attributes['textAlign'] ?? '';

// Don't render an empty CTA -- avoids blank space in the frontend.
if ( empty( $heading ) && empty( $description ) && empty( $button_text ) ) {
	return '';
}

$align_class = $text_align ? ' has-text-align-' . esc_attr( $text_align ) : '';

// get_block_wrapper_attributes() is the PHP equivalent of useBlockProps().
$wrapper_attributes = get_block_wrapper_attributes( [
	'class' => $align_class,
] );

$rel = $open_new_tab ? 'noopener noreferrer' : '';
?>
<div <?php echo $wrapper_attributes; ?>>
	<?php if ( $media_url ) : ?>
		<div
			class="wp-block-my-plugin-call-to-action__media"
			style="background-image:url(<?php echo esc_url( $media_url ); ?>)"
			role="img"
			aria-label="<?php echo esc_attr( $media_alt ); ?>"
		></div>
	<?php endif; ?>

	<div class="wp-block-my-plugin-call-to-action__content">
		<?php if ( $heading ) : ?>
			<h2 class="wp-block-my-plugin-call-to-action__heading">
				<?php echo wp_kses_post( $heading ); ?>
			</h2>
		<?php endif; ?>

		<?php if ( $description ) : ?>
			<p class="wp-block-my-plugin-call-to-action__description">
				<?php echo wp_kses_post( $description ); ?>
			</p>
		<?php endif; ?>

		<?php if ( $button_text ) : ?>
			<?php if ( $button_url ) : ?>
				<a
					class="wp-block-my-plugin-call-to-action__button"
					href="<?php echo esc_url( $button_url ); ?>"
					<?php echo $open_new_tab ? 'target="_blank"' : ''; ?>
					<?php echo $rel ? 'rel="' . esc_attr( $rel ) . '"' : ''; ?>
				>
					<?php echo wp_kses_post( $button_text ); ?>
				</a>
			<?php else : ?>
				<span class="wp-block-my-plugin-call-to-action__button">
					<?php echo wp_kses_post( $button_text ); ?>
				</span>
			<?php endif; ?>
		<?php endif; ?>
	</div>
</div>
```

## Key Decisions

| Decision | Rationale |
|----------|-----------|
| RichText for all text fields | Three-tier hierarchy: content editing belongs on the canvas (tier 1), not in sidebar TextControls. Direct manipulation is the primary interface. |
| `allowedFormats` restricted on heading | Headings should not contain links or code spans. Unrestricted RichText would let editors add formats that break semantics. |
| `allowedFormats={ [] }` on button text | Button labels are always plain text. Inline formatting (bold, italic, links) would break the button's visual design and create nested interactive elements. |
| Toolbar for text alignment, not `typography.textAlign` support | Alignment is a frequent structural action -- it belongs in the toolbar (tier 2). The supports-based control would place it in the sidebar typography panel (tier 3), requiring an extra click. |
| PanelBody not ToolsPanel | Only 2 required settings (URL, new tab). ToolsPanel's progressive disclosure adds complexity without benefit for small control sets. |
| `<span>` for button in editor, `<a>` in render.php | Clicking an `<a>` in the editor would navigate away. Rendering as `<span>` in edit mode prevents accidental navigation while keeping the same visual appearance. |
| MediaPlaceholder on canvas, not sidebar | Image selection is a primary action for this block. MediaPlaceholder in the canvas (tier 1) provides drag-drop, library, and URL -- all discoverable without opening the sidebar. |
| `role="img"` + `aria-label` on background div | Background images via CSS are invisible to screen readers. ARIA attributes restore accessibility. |
| `color-mix()` for overlay and hover | Modern Gutenberg pattern from polish-patterns.md. Adapts to any theme's contrast color without hardcoding opacity layers. |
| Focus-visible ring with transparent outline | Matches Gutenberg's `button-style-outset__focus` mixin. The transparent outline is a Windows High Contrast Mode fallback. |
