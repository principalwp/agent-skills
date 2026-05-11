# Safe `wp search-replace`

Use this file when migrating domains, switching http→https, or changing paths.

## Recommended workflow

1. Backup:
   - `wp db export`
2. Dry run:
   - `wp search-replace OLD NEW --dry-run`
3. Run for real (carefully choose scope):
   - consider `--all-tables-with-prefix` if you need to include non-core tables with the WP prefix
4. Flush:
   - `wp cache flush`
   - `wp rewrite flush`

## Multisite notes

For multisite, decide whether you're replacing:

- a single site (`--url=...`), or
- across the network (`--network` or iterating `wp site list`).

Read:
- `references/multisite.md`

## Common flags

- `--dry-run`
- `--precise` (PHP path instead of SQL — slower but handles serialized/escaped/base64 cases)
- `--skip-columns=...` (avoid touching large/binary columns)
- `--report-changed-only`
- `--export=<file>` (write the replacement as SQL instead of mutating the DB live)
- `--log --before_context=40 --after_context=40` (audit trail of every replacement)

## Canonical safe recipe

```bash
wp search-replace OLD NEW \
    --dry-run \
    --precise \
    --skip-columns=guid \
    --report-changed-only
```

Drop `--dry-run` once the report looks right. Re-run with `--dry-run` afterwards and confirm 0 replacements — see "Multiple passes" below.

## Traps that bite real migrations

### `--skip-columns=guid` is NOT default

Always pass `--skip-columns=guid`. WordPress documentation explicitly states the `guid` column should never change once a post is published — it's a stable identifier used by RSS readers to dedupe items. Replacing `guid` causes every existing reader to re-show every old post as new. There is no automatic skip; you have to add it yourself every time.

### `--precise` is misnamed

The flag's name implies "more accurate," but it actually forces PHP-side traversal instead of `SQL REPLACE()`. It's ~15–20× slower. Use it when the data contains:

- PHP-serialized arrays/objects with length-prefixed strings (the SQL path can break length prefixes)
- HTML where the search string appears in escaped form (`&lt;`, `<`, etc.)
- Base64-encoded blobs that may contain the search string after decoding

For plain `wp_posts.post_content` URL replacements, the default SQL path is fine.

### Serialized array KEYS are not replaced

`--recurse-objects` (default `true`) walks values, not keys. Renaming a widget `sidebar-1` → `my-sidebar-1` inside `theme_mods_*` won't work via `search-replace` — the count says "X replacements made" because matching values in the same row matched, but the structural key rename never happens. For key renames, write a `wp eval` script that unserializes, mutates, and re-saves.

### `--network` does NOT touch multisite-management tables

`wp search-replace ... --network` skips `wp_blogs`, `wp_site`, `wp_blogmeta`, `wp_sitemeta`. The `domain` column in `wp_blogs` stays at the old hostname after a multisite migration. Use `--all-tables` (or hit those tables with raw SQL) to fix multisite domain entries.

### `siteurl`/`home` typos are accepted silently

A typo like `http;//` (semicolon for colon) passes through without validation, the replacement reports "Success", and the next page load throws a fatal `ValueError` from `setcookie()` deep inside another plugin. Always:

- `--dry-run` first
- After a real run, re-fetch via `wp option get siteurl` and `wp option get home` and confirm they're well-formed URLs

### One pass is sometimes not enough

Same DB, same OS family — sometimes finishes in one pass, sometimes takes 7. Usually traced to `utf8mb4` collation differences or PCRE/serialized-length recalculation. **Always re-run** the same `--dry-run` after the real run; if it reports more replacements, run for real again. Loop until the dry run reports 0 replacements.

### `--export=<file>` for safer migrations

For production migrations, prefer:

```bash
wp search-replace OLD NEW --export=migrate.sql --all-tables-with-prefix
```

This writes the replacement as SQL (a modified dump) instead of mutating the live DB. Apply it through your normal deploy pipeline with rollback in place.
