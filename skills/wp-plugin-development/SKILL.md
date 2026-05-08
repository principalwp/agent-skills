---
name: wp-plugin-development
description: "Use when developing WordPress plugins: architecture and hooks, activation/deactivation/uninstall, admin UI and Settings API, data storage, cron/tasks, security (nonces/capabilities/sanitization/escaping), and release packaging."
compatibility: "Targets WordPress 6.9+ (PHP 7.2.24+). Filesystem-based agent with bash + node. Some workflows require WP-CLI."
---

# WP Plugin Development

## When to use

Use this skill for plugin work such as:

- creating or refactoring plugin structure (bootstrap, includes, namespaces/classes)
- adding hooks/actions/filters
- activation/deactivation/uninstall behavior and migrations
- adding settings pages / options / admin UI (Settings API)
- security fixes (nonces, capabilities, sanitization/escaping, SQL safety)
- packaging a release (build artifacts, readme, assets)

## Inputs required

- Repo root + target plugin(s) (path to plugin main file if known).
- Where this plugin runs: single site vs multisite; WP.com conventions if applicable.
- Target WordPress + PHP versions (affects available APIs and placeholder support in `$wpdb->prepare()`).

## Procedure

### 0) Triage and locate plugin entrypoints

1. Run triage:
   - `node skills/wp-project-triage/scripts/detect_wp_project.mjs`
2. Detect plugin headers (deterministic scan):
   - `node skills/wp-plugin-development/scripts/detect_plugins.mjs`

If this is a full site repo, pick the specific plugin under `wp-content/plugins/` or `mu-plugins/` before changing code.

### 1) Follow a predictable architecture

Guidelines:

- Keep a single bootstrap (main plugin file with header).
- Avoid heavy side effects at file load time; load on hooks.
- Prefer a dedicated loader/class to register hooks.
- Keep admin-only code behind `is_admin()` (or admin hooks) to reduce frontend overhead.
- Register all hooks (`add_action`/`add_filter`) at plugin load time —
  in the constructor, a static `init()` method, or `plugins_loaded`. If
  `add_action()` is called inside a method that itself runs on `init`,
  a new copy of that callback is registered on every request, causing the
  callback to execute multiple times when its hook fires.
- When a class has both admin-only hooks (menus, AJAX, settings) and
  cron/frontend hooks, split registration into two methods: one called
  unconditionally for cron/frontend callbacks, one gated behind
  `is_admin()` for admin-only hooks.

See:
- `references/structure.md`

When registering a CPT, match `publicly_queryable` to the feature's intent.
If requirements describe a "landing page", "front-end URL", or "page for each
{resource}", the CPT needs `publicly_queryable => true` (or an explicit
alternative like a block inserted into manually-created pages, documented as a
design decision with rationale). Mismatched queryability is a common source of
"where's my page?" bugs.

### 2) Hooks and lifecycle (activation/deactivation/uninstall)

Activation hooks are fragile; follow guardrails:

- register activation/deactivation hooks at top-level, not inside other hooks
- flush rewrite rules only when needed and only after registering CPTs/rules
- uninstall should be explicit and safe (`uninstall.php` or `register_uninstall_hook`)

See:
- `references/lifecycle.md`

### 3) Settings and admin UI (Settings API)

Prefer Settings API for options:

- `register_setting()`, `add_settings_section()`, `add_settings_field()`
- sanitize via `sanitize_callback`

See:
- `references/settings-api.md`

### 4) Security baseline (always)

Before shipping:

- Validate/sanitize input early; escape output late.
- Use nonces to prevent CSRF *and* capability checks for authorization.
- Avoid directly trusting `$_POST` / `$_GET`; use `wp_unslash()` and specific keys.
- Use `$wpdb->prepare()` for SQL; avoid building SQL with string concatenation.
- Always enforce constraints server-side in save/update handlers (min/max
  counts, required fields, format rules). Client-side validation improves UX
  and should be kept, but is not a security boundary — direct POST requests
  and API calls bypass JavaScript entirely.

See:
- `references/security.md`
- `references/security-deep-cuts.md` (nonce internals, password hashing, capability edge cases, KSES defaults)

### 5) Data storage, cron, migrations (if needed)

- Prefer options for small config; custom tables only if necessary.
- API keys, secrets, and credentials must be stored in environment variables
  or a host-provided secrets store (typically accessed via PHP constants
  defined outside the WordPress database). Never store secrets in
  `wp_options`, post meta, or transients.
- For cron tasks, ensure idempotency and provide manual run paths (WP-CLI or admin).
- For schema changes, write upgrade routines and store schema version.

See:
- `references/data-and-cron.md`

### 6) Avoid common WordPress gotchas

Before shipping, review `references/gotchas-and-traps.md` for return-value traps and common misconceptions. Key traps:

- `get_post_meta($id, 'key', true)` returns `''` on missing key, not `null`
- `$wpdb->insert()` returns rows affected, not the insert ID — use `$wpdb->insert_id`
- `update_post_meta()` returns `false` when value is identical (indistinguishable from failure)
- `is_admin()` checks URL path, not user role
- Neither `wp_redirect()` nor `wp_safe_redirect()` calls `exit`

See:
- `references/gotchas-and-traps.md` (return values, numeric limits, hook ordering, query differences)
- `references/i18n-gotchas.md` (translation format, text domains, locale functions)

## Verification

- Plugin activates with no fatals/notices.
- Settings save and read correctly (capability + nonce enforced).
- Uninstall removes intended data (and nothing else).
- Run repo lint/tests (PHPUnit/PHPCS if present) and any JS build steps if the plugin ships assets.

## Failure modes / debugging

- Activation hook not firing:
  - hook registered incorrectly (not in main file scope), wrong main file path, or plugin is network-activated
- Settings not saving:
  - settings not registered, wrong option group, missing capability, nonce failure
- Security regressions:
  - nonce present but missing capability checks; or sanitized input not escaped on output

See:
- `references/debugging.md`

## Escalation

For canonical detail, consult the Plugin Handbook and security guidelines before inventing patterns.
