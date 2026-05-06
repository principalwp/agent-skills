# Tooling and testing

Use this file when deciding what commands to run and what “good verification” looks like.

## Common toolchains

- `@wordpress/scripts` for build/lint/test:
  - https://developer.wordpress.org/block-editor/reference-guides/packages/packages-scripts/
- `@wordpress/create-block` to scaffold new blocks:
  - https://developer.wordpress.org/block-editor/reference-guides/packages/packages-create-block/
- Interactivity API template for `create-block`:
  - https://www.npmjs.com/package/@wordpress/create-block-interactive-template
- `@wordpress/env` (wp-env) for local WordPress environments (alternative to WP Playground):
  - https://developer.wordpress.org/block-editor/reference-guides/packages/packages-env/

## Verification checklist

- `npm run build` (or repo equivalent) succeeds.
- JS lint passes (repo-specific).
- E2E tests pass if present.
- Manual: insert block, save post, reload editor, confirm no “Invalid block”.

## Customizing the @wordpress/scripts build

Default `wp-scripts build` auto-discovers `src/**/block.json` entries. Override via `webpack.config.js` at plugin root when you need JS-only bundles (variations, admin enhancements, format types) alongside blocks:

```js
// webpack.config.js
const defaultConfig = require( '@wordpress/scripts/config/webpack.config' );
const path = require( 'path' );

module.exports = {
    ...defaultConfig,
    entry: {
        ...defaultConfig.entry(),
        'variations/index': path.resolve( __dirname, 'src/variations/index.js' ),
        'admin/index':      path.resolve( __dirname, 'src/admin/index.js' ),
    },
};
```

Rules:
- `defaultConfig.entry` is a **function** — call it to get auto-discovered block entries, then spread. Do not assign the function itself to `entry`.
- Output goes to `build/` mirroring entry keys (`build/variations/index.js`, `build/variations/index.asset.php`).
- Enqueue extras via the generated `asset.php`:

  ```php
  $asset = include plugin_dir_path( __FILE__ ) . 'build/variations/index.asset.php';
  wp_enqueue_script(
      'my-plugin-variations',
      plugins_url( 'build/variations/index.js', __FILE__ ),
      $asset['dependencies'],
      $asset['version'],
      true
  );
  ```

- Never hand-list `@wordpress/*` dependencies — `DependencyExtractionWebpackPlugin` (bundled in `@wordpress/scripts`) generates them and maps them to the right WP script handles.
- When merging additional webpack rules (e.g., SVG as React components), preserve the default `module.rules` — they already handle CSS/SCSS/images. Spread, don't replace.
- After every build, confirm: `build/*/block.json`, `build/*/index.asset.php`, and all extra entries exist.
