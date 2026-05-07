# Cron, caches, and rewrites

Use this file when debugging background jobs or "changes not visible".

## Inspect cron state

```bash
# What's queued?
wp cron event list --fields=hook,next_run_relative,recurrence

# Schedules registered (Daily, Twice Daily, etc.)
wp cron schedule list
```

## Run events

```bash
# Run all events that are due NOW (synchronously, in this process)
wp cron event run --due-now

# Run a specific event by name
wp cron event run my_plugin_hourly_task

# Re-run an event by ID (when multiple instances of the same hook are queued)
wp cron event run my_plugin_hourly_task --due-now
```

There is **no `--all` flag**. The canonical "fire everything pending" is `--due-now`. Forces fatals to surface in the terminal that would otherwise be silent on the web side (WP's spawn-loopback request swallows them).

## The `doing_cron` lock trap

If a previous cron run crashed and left the `doing_cron` transient set, `wp cron event run --due-now` skips events to "prevent overlapping runs." It manifests as:

```
Success: Executed a total of 0 events.
```

…with no explanation. Fix:

```bash
wp transient delete doing_cron
wp cron event run --due-now
```

For ops scripts that assume `wp cron event run --due-now` is idempotent, add the `delete doing_cron` step at the top.

## `wp cron test` — unreliable behind proxies

`wp cron test` issues a self-HTTP request to verify cron can spawn. It returns non-200 for many legitimate setups:

- nginx reverse proxies that don't route loopback back to the application
- container networks where the WP container can't reach itself by external hostname
- hosts that block loopback HTTP entirely (Pantheon, WP Engine, Kinsta, VIP often do)

If scheduled cron via system crontab works fine, **don't gate deploys on `wp cron test`** — it's checking the WP-Cron HTTP-spawn mechanism, not your actual scheduler.

## `DISABLE_WP_CRON` + system cron pattern

Standard production setup: disable WP-Cron HTTP spawning, run via system cron:

```php
// wp-config.php
define( 'DISABLE_WP_CRON', true );
```

```cron
# crontab — every 5 minutes
*/5 * * * * cd /var/www/html && /usr/local/bin/wp cron event run --due-now --quiet
```

`--quiet` suppresses the per-event stdout so cron mail isn't noisy. Errors and `WP_CLI::error()` still go to stderr.

## Cache + rewrite

```bash
# Flush object cache
wp cache flush

# Flush rewrite rules
wp rewrite flush
```

### Diagnosing transient storage

```bash
wp transient type
```

Tells you whether transients route to the persistent object cache (Redis/Memcached drop-in) or to the database. If you've installed Redis but `wp transient type` reports DB-backed, the drop-in isn't loaded — investigate `wp-content/object-cache.php`.

```bash
# Bulk-clean only expired transients (safe-ish)
wp transient delete --expired

# Bulk-clean ALL transients (cache-stampede risk on busy sites)
wp transient delete --all
```

## Guardrails

- Don't `--due-now` blindly on production without understanding what's queued — inspect first.
- `wp cache flush` on a busy site can cause a load spike (every request rebuilds caches simultaneously); coordinate during low-traffic windows.
- `wp rewrite flush` writes to `.htaccess` on Apache; ensure the file is writable.
