# Security deep cuts

## Nonce verification return value

`wp_verify_nonce()` does **not** return `true` on success. It returns:

- `1` — nonce created 0–12 hours ago
- `2` — nonce created 12–24 hours ago
- `false` — invalid or expired

Code like `if ( true === wp_verify_nonce( $n, $action ) )` always fails. Use `if ( wp_verify_nonce( $n, $action ) )` (truthy check) or `=== false` for the failure path.

`check_ajax_referer()` checks `_ajax_nonce` first, then falls back to `_wpnonce` — useful when migrating between AJAX and form-post handlers.

## Escaping function filter names (legacy)

The filter names don't match the function names — surprises anyone trying to wp_unfilter or remove a filter:

| Function | Filter name |
|----------|-------------|
| `esc_url()` | `clean_url` |
| `esc_attr()` | `attribute_escape` |
| `esc_js()` | `js_escape` |

## esc_url vs esc_url_raw

- `esc_url()` — encodes `&`, `'`, and other chars. Use for **display** in HTML.
- `esc_url_raw()` — does NOT encode. Use for **database storage** and `wp_redirect()`.

Storing the output of `esc_url()` and then displaying it again double-encodes (`&amp;amp;`). Passing the output of `esc_url()` to `wp_redirect()` produces a broken URL because `&amp;` becomes a literal in the Location header.

## wp_check_filetype

`wp_check_filetype()` validates **extension only**, never inspects file content. A `.jpg` file containing PHP will pass `wp_check_filetype()` happily. For actual content validation, use `wp_check_filetype_and_ext()`, which checks magic bytes via `finfo`.

## Protocol allowlist

`javascript` protocol is **deliberately excluded** from `wp_allowed_protocols()` — XSS prevention. If you build a URL via user input and pass it through `esc_url()`, javascript-scheme links are stripped silently. Don't add `javascript` back to the allowlist "to make a feature work" without understanding what XSS surface that opens.

## KSES defaults

Default `$allowedtags` (the array used by bare `wp_kses()` / `wp_kses_data()`) includes only: `a`, `abbr`, `acronym`, `b`, `blockquote`, `cite`, `code`, `del`, `em`, `i`, `q`, `s`, `strike`, `strong`. Notably **no `<img>`** — running unfiltered HTML through bare `wp_kses()` strips images, which surprises developers expecting "wp_kses with defaults" to behave like Gmail-comment levels of permissiveness.

`wp_kses_post()` uses the `'post'` context — much wider allowlist (images, forms, embeds). Use it when you want post-content-equivalent permissiveness.

## sanitize_text_field vs sanitize_textarea_field

- `sanitize_text_field()` — **strips newlines**.
- `sanitize_textarea_field()` — **preserves newlines**.

The naming suggests "text vs textarea" is about UI, but the actual difference is line-break handling. Running multi-line user input through `sanitize_text_field()` silently flattens it.

## wp_redirect vs wp_safe_redirect

- `wp_safe_redirect()` validates against the `allowed_redirect_hosts` allowlist.
- **Neither calls `exit`** — you must call `exit` (or `die`) yourself.

A `wp_redirect()` followed by no `exit` keeps executing — including writing more headers, running shutdown handlers, and emitting body content — leading to "Headers already sent" errors and security gaps where redirected requests still execute privileged code.

## Capabilities deep cuts

| Capability | Behavior |
|------------|----------|
| `do_not_allow` | Always fails — **even for super admins**. Standard "deny" capability for `map_meta_cap` filters. |
| `DISALLOW_UNFILTERED_HTML` | When the constant is true, the `unfiltered_html` cap maps to `do_not_allow` for ALL users including super admins. |
| `edit_user` (on own profile) | Requires nothing (empty caps) — any authenticated user can edit themselves. Don't rely on `current_user_can('edit_user', $id)` alone for self-edit gates. |

## $wpdb->prepare format specifiers

`%s` (string), `%d` (integer), `%f` (float), `%i` (backtick-wrapped identifier — for table/column names).

`%i` (added in WP 6.2) is the safe way to interpolate table/column names. Before it, the only options were dynamic SQL with `$wpdb->prefix . 'mytable'` (still safe for trusted prefix) or unsafe string concatenation with user input. If you see code building table names by string concatenation with anything that could be user-influenced, that's an injection vector — replace with `%i`.
