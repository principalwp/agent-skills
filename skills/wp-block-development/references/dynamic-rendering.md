# Dynamic blocks (server rendering)

Use this file when converting a block to dynamic, or debugging frontend output mismatch.

## Choose the mechanism

- Prefer `render` in `block.json` (dynamic render file).
- Alternative: pass `render_callback` when registering the block in PHP.

## Wrapper attributes

In PHP render output, always use:

- `get_block_wrapper_attributes()`

This preserves support-generated classes/styles.

## ServerSideRender attribute type coercion

`<ServerSideRender>` passes block attributes as URL query parameters. PHP
receives everything as strings. This means `false` arrives as `"false"`,
which is truthy in PHP (`boolval("false") === true`, `!empty("false") === true`).
Numeric values arrive as strings too (`3` becomes `"3"`).

**Rule:** In `render.php`, always use `filter_var()` to sanitize attributes
that aren't plain strings:

- **Booleans:** `filter_var( $value, FILTER_VALIDATE_BOOLEAN )` — correctly
  handles `"false"`, `"0"`, `""`, `"no"`, `"off"`.
- **Integers:** `filter_var( $value, FILTER_VALIDATE_INT, ['options' => ['default' => $fallback]] )`
  — rejects non-numeric strings instead of silently returning 0.
- **Floats:** `filter_var( $value, FILTER_VALIDATE_FLOAT )`.

Never use `boolval()`, `!empty()`, or loose casts on block attributes in
`render.php`. Front-end renders receive properly typed values from the block
parser, but `ServerSideRender` does not — sanitize defensively for both paths.

## Practical checklist

- Ensure PHP file exists and is reachable from the block root.
- Ensure registration runs on every request (not only in admin).
- Keep `save()` empty or `null` for fully dynamic output, unless you intentionally save fallback markup.

