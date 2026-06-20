---
name: wp-block-development
description: "Use when developing WordPress (Gutenberg) blocks: block.json metadata, register_block_type(_from_metadata), attributes/serialization, supports, dynamic rendering (render.php/render_callback), deprecations/migrations, viewScript vs viewScriptModule, @wordpress/scripts/@wordpress/create-block build and test workflows, and inline content within paragraphs (registerFormatType, RichText format API)."
compatibility: "Targets WordPress 6.9+ (PHP 7.2.24+). Filesystem-based agent with bash + node. Some workflows require WP-CLI."
---

# WP Block Development

## When to use

Use this skill for block work such as:

- creating a new block, or updating an existing one
- changing `block.json` (scripts/styles/supports/attributes/render/viewScriptModule)
- fixing “block invalid / not saving / attributes not persisting”
- adding dynamic rendering (`render.php` / `render_callback`)
- block deprecations and migrations (`deprecated` versions)
- build tooling for blocks (`@wordpress/scripts`, `@wordpress/create-block`, `wp-env`)
- building inline content within paragraphs (`registerFormatType`, RichText formats)

## Inputs required

- Repo root and target (plugin vs theme vs full site).
- The block name/namespace and where it lives (path to `block.json` if known).
- Target WordPress version range (especially if using modules / `viewScriptModule`).

## Procedure

### 0) Triage and locate blocks

1. Run triage:
   - `node skills/wp-project-triage/scripts/detect_wp_project.mjs`
2. List blocks (deterministic scan):
   - `node skills/wp-block-development/scripts/list_blocks.mjs`
3. Identify the block root (directory containing `block.json`) you’re changing.

If this repo is a full site (`wp-content/` present), be explicit about *which* plugin/theme contains the block.

### 1) Check for inline requirements FIRST

Before creating any block, check whether the requirement asks for content that
flows **inline within paragraph text** (e.g., "inline badge," "widget inside a
sentence," "inline weather display"). If it does, **do not register a block** —
use the RichText Format API instead.

Read:
- `references/inline-and-format-types.md`

If the requirement is for a block-level element (its own line in the editor),
continue to step 2.

### 2) Create a new block (if needed)

If you are creating a new block, prefer scaffolding rather than hand-rolling structure:

- Use `@wordpress/create-block` to scaffold a modern block/plugin setup.
- If you need Interactivity API from day 1, use the interactive template.

Read:
- `references/creating-new-blocks.md`

After scaffolding:

1. Re-run the block list script and confirm the new block root.
2. Continue with the remaining steps (model choice, metadata, registration, serialization).

### 3) Ensure apiVersion 3 (WordPress 6.9+)

WordPress 6.9 enforces `apiVersion: 3` in the block.json schema. Blocks with apiVersion 2 or lower trigger console warnings when `SCRIPT_DEBUG` is enabled.

**Why this matters:**
- WordPress 7.0 will run the post editor in an iframe regardless of block apiVersion.
- apiVersion 3 ensures your block works correctly inside the iframed editor (style isolation, viewport units, media queries).

**Migration:** Changing from version 2 to 3 is usually as simple as updating the `apiVersion` field in `block.json`. However:
- Test in a local environment with the iframe editor enabled.
- Ensure any style handles are included in `block.json` (styles missing from the iframe won't apply).
- Third-party scripts attached to a specific `window` may have scoping issues.

Read:
- `references/block-json.md` (apiVersion and schema details)

### 4) Pick the right block model

- **Static block** (markup saved into post content): implement `save()`; keep attributes serialization stable.
- **Dynamic block** (server-rendered): use `render` in `block.json` (or `render_callback` in PHP) and keep `save()` minimal or `null`.
- **Interactive frontend behavior**:
  - Prefer `viewScriptModule` for modern module-based view scripts where supported.
  - If you're working primarily on `data-wp-*` directives or stores, also use `wp-interactivity-api`.

### 5) Update `block.json` safely

Make changes in the block’s `block.json`, then confirm registration matches metadata.

For field-by-field guidance, read:
- `references/block-json.md`

Common pitfalls:

- changing `name` breaks compatibility (treat it as stable API)
- changing saved markup without adding `deprecated` causes “Invalid block”
- adding attributes without defining source/serialization correctly causes “attribute not saving”

### 6) Register the block (server-side preferred)

Prefer PHP registration using metadata, especially when:

- you need dynamic rendering
- you need translations (`wp_set_script_translations`)
- you need conditional asset loading

Read and apply:
- `references/registration.md`

### 7) Implement edit/save/render patterns

Follow wrapper attribute best practices:

- Editor: `useBlockProps()`
- Static save: `useBlockProps.save()`
- Dynamic render (PHP): `get_block_wrapper_attributes()`

If the block reads per-post data (content, meta, title, excerpt, etc.), read
the context reference to ensure the editor component works inside Query Loop:
- `references/context-and-query-loop.md`

Read:
- `references/supports-and-wrappers.md`
- `references/dynamic-rendering.md` (if dynamic)

### 8) Inner blocks (block composition)

If your block is a “container” that nests other blocks, treat Inner Blocks as a first-class feature:

- Use `useInnerBlocksProps()` to integrate inner blocks with wrapper props.
- Keep migrations in mind if you change inner markup.

Read:
- `references/inner-blocks.md`

### 9) Attributes and serialization

Before changing attributes:

- confirm where the attribute value lives (comment delimiter vs HTML vs context)
- avoid the deprecated `meta` attribute source

Read:
- `references/attributes-and-serialization.md`

### 10) Migrations and deprecations (avoid "Invalid block")

If you change saved markup or attributes:

1. Add a `deprecated` entry (newest → oldest).
2. Provide `save` for old versions and an optional `migrate` to normalize attributes.

Read:
- `references/deprecations.md`

### 11) Tooling and verification commands

Prefer whatever the repo already uses:

- `@wordpress/scripts` (common) → run existing npm scripts
- WP Playground (preferred) or `wp-env` → use for local WP + E2E

Read:
- `references/tooling-and-testing.md`

### 12) Editor data fetching patterns

When a block’s `edit.js` fetches data (REST endpoints, `apiFetch`, external
services), follow these patterns:

**Three-state rendering**: Every data-fetching component must handle loading,
success, and error states explicitly:
```jsx
const [ data, setData ] = useState( null );
const [ isLoading, setIsLoading ] = useState( false );
const [ error, setError ] = useState( null );

// In render:
if ( isLoading ) return <Spinner />;
if ( error ) return <Notice status=”error”>{ error }</Notice>;
if ( ! data ) return <Placeholder>...</Placeholder>;
```

**Visible error handling**: Never silently swallow fetch errors. Every `catch`
block must set an error state that renders user-visible feedback:
- BAD: `catch (_e) { setResults([]); }` — user sees empty results, no
  explanation
- GOOD: `catch (err) { setError( err.message ); }` — user sees what went wrong

**Endpoint verification**: When calling custom REST endpoints via `apiFetch`,
verify the endpoint path and method match the PHP `register_rest_route()`
registration. Common mismatches:
- Path prefix: `/wp/v2/` (core) vs `/my-plugin/v1/` (custom)
- Method: `GET` vs `POST` (especially for search endpoints with body params)
- Capability: editor needs `edit_posts` but endpoint requires `manage_options`

**Testing strategy for data fetching** (all E2E via Playwright + WP Playground):
- Test REST endpoint response shape via `page.request.get()`
- Test permission gates by authenticating as different user roles
- Test error states by intercepting API calls with `page.route()` or Blueprint
  `runPHP` steps injecting `pre_http_request` filters
- Test all three component states (loading, success, error) by navigating to
  the block on the frontend under each condition

## Verification

- Block appears in inserter and inserts successfully.
- Saving + reloading does not create “Invalid block”.
- Frontend output matches expectations (static: saved markup; dynamic: server output).
- Assets load where expected (editor vs frontend).
- Run the repo’s lint/build/tests that triage recommends.

## Failure modes / debugging

If something fails, start here:

- `references/debugging.md` (common failures + fastest checks)
- `references/attributes-and-serialization.md` (attributes not saving)
- `references/deprecations.md` (invalid block after change)
- `references/inline-and-format-types.md` (block renders with line breaks instead of inline — wrong API choice)

## Deep reference

For non-obvious block system behaviors (naming regex, global attributes, render internals, block hooks scope, HTML API patterns, block bindings, editor data stores), see:
- `references/block-api-internals.md`
- `references/editor-data-stores.md`
- `references/extension-points.md` — block variations, `editor.BlockEdit` / `blocks.registerBlockType` filter hooks, block transforms (use when extending core blocks or touching many blocks at once)

## Escalation

If you’re uncertain about upstream behavior/version support, consult canonical docs first:

- WordPress Developer Resources (Block Editor Handbook, Theme Handbook, Plugin Handbook)
- Gutenberg repo docs for bleeding-edge behaviors
