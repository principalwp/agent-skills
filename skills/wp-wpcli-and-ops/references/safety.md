# Safety rules (WP-CLI)

Use this file before running any write operations.

## Golden rules

- Assume production is **unsafe** unless explicitly confirmed.
- Always confirm targeting:
  - `--path` (WordPress root)
  - `--url` (multisite / specific site targeting)
- Prefer a backup (`wp db export`) before risky operations.
- Prefer `--dry-run` where available (especially `search-replace`).

## High-risk commands (require explicit confirmation)

- `wp db reset`
- `wp db import` (overwrites data)
- `wp search-replace` (can affect serialized data and URLs)
- bulk deletes (`wp post delete --force --all`, `wp user delete --reassign`, etc.)
- plugin/theme mass updates on production

## Plugin/theme updates with `--version` are non-atomic

`wp plugin update foo --version=1.2.3` does:

1. Delete the existing plugin directory
2. Download the requested version
3. Install

If step 2 fails (network blip, paid plugin auth issue, server-side rate limit), step 1 has already happened — you're left with no plugin and possibly de-activated. There is no rollback.

In production, always wrap version-pinned updates with maintenance mode:

```bash
wp maintenance-mode activate
wp plugin update foo --version=1.2.3 || wp maintenance-mode deactivate
wp plugin verify-checksums foo --strict   # confirm the install is intact
wp maintenance-mode deactivate
```

## `verify-checksums` after risky updates

After any plugin/theme/core update, run:

```bash
wp core verify-checksums --strict
wp plugin verify-checksums --all --strict
```

`--strict` flags even `readme.txt` drift; pair with `--exclude=<files>` for known-good drift you've reviewed. This catches partial-install corruption and is also a lightweight intrusion-detection check.

## Memory limit on long-running ops

WP-CLI sets `ini_set('memory_limit', -1)` at runtime regardless of `php.ini`, `WP_MEMORY_LIMIT`, or `WP_MAX_MEMORY_LIMIT`. A long-running command (search-replace on a huge DB, mass post operation) that should fail fast instead consumes all RAM and gets OOM-killed by the kernel — leaving no shutdown hooks, no usable error, no progress indicator. Some ops will appear to hang and then die.

Mitigations:

- For production cron-style runs, cap memory explicitly via env: `WP_CLI_PHP_ARGS='-d memory_limit=1G' wp <command>`
- Or re-set `memory_limit` from a hook on `after_wp_load`:
  ```php
  WP_CLI::add_hook( 'after_wp_load', function() {
      ini_set( 'memory_limit', '512M' );
  } );
  ```
  loaded via `--require=cap-memory.php` or `WP_CLI_REQUIRE`.

## Composer autoloader collisions (Bedrock / Roots)

When a site uses Composer in `wp-content/` (Bedrock, Roots, or any custom mu-plugins/vendor setup), WP-CLI's bundled autoloader can prevent site classes from loading. Symptom: "Class not found" errors in CLI but the same code works fine in the browser.

Workaround:

```bash
WP_CLI_EARLY_REQUIRE=/path/to/site/vendor/autoload.php wp <command>
```

Or set in `wp-cli.yml`:

```yaml
require:
  - vendor/autoload.php
```

## Logging

For ops scripts, log:

- date/time
- environment (dev/staging/prod)
- exact WP-CLI commands
- exit codes

Pipe through `tee` so the trail survives even if the terminal is lost:

```bash
wp import file.wxr 2>&1 | tee import-$(date +%Y%m%d-%H%M).log
```
