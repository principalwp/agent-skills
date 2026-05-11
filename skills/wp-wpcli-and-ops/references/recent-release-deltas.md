# WP-CLI release deltas (last 12 months)

Concrete deltas from WP-CLI core 2.12.0 (May 2025) and the package commands that shipped in late 2025 / Q1-Q2 2026. Anchored to release notes and `make.wordpress.org/cli`.

WP-CLI core itself only released **2.12.0** in the window; the surface-area changes since happened in package commands, which release independently.

## WP-CLI 2.12.0 establishes the package floor

`composer require wp-cli/wp-cli ^2.12` is now required by every package release in the next 12 months. 2.12 ships:

- PHP 8.4 cleanup (implicit-nullable parameter fixes, `E_STRICT` removed, Requests bumped to 2.0.12)
- MariaDB vs MySQL server detection
- Soft failure when no `mysql` binary is present (no longer hard errors — important for distroless containers)
- New `WP_CLI_REQUIRE` env var (loads after WP, vs `WP_CLI_EARLY_REQUIRE` before)
- `WP_CLI_EARLY_REQUIRE` now accepts multiple files
- Configurable user-agent for firewall log identification
- Command suggester covers taxonomies and post types

**8+ packages already require WP-CLI 2.13** (not yet released). Watch composer resolutions if you pin WP-CLI.

Source: [WP-CLI 2.12.0 release notes](https://make.wordpress.org/cli/2025/05/07/wp-cli-v2-12-0-release-notes/)

## `wp post list` accepts JSON for `--tax_query`, `--meta_query`, `--post_date` (2.12.0)

Replaces the brittle `--tax_query__0__taxonomy=...` style.

```bash
wp post list \
  --tax_query='{"taxonomy":"category","terms":[3,5]}' \
  --meta_query='{"key":"_priority","compare":">","value":5,"type":"NUMERIC"}'
```

`wp post create --tax_input='{"category":[3,5],"post_tag":["news"]}'` (entity-command 2.8.6) — assign hierarchical taxonomy terms with IDs in one call.

## `wp post meta get --single` default change (2.12.0)

`--single=true` is the new default. Scripts that relied on getting an array back from a multi-value meta key now get the first scalar instead. Audit any script reading meta — easy silent break.

```bash
wp post meta get 42 my_key                # single scalar by default
wp post meta get 42 my_key --single=false # array of all values
```

## `wp config add` / `wp config update` — explicit add-only / update-only (config-command 2.5.0)

Closes a real gap. Previously `wp config set` did both add and overwrite — no way to fail-on-duplicate or fail-on-missing in idempotent provisioning.

```bash
wp config add  WP_DEBUG true --raw    # errors if WP_DEBUG already exists
wp config update WP_DEBUG true --raw  # errors if WP_DEBUG does NOT exist
```

Behavior change: `wp config get` of booleans now prints literal `true`/`false` instead of `1`/empty. Any script grepping `=1` for booleans breaks. `dbpass` is now treated as sensitive in output. `wp config create` no longer mis-detects a parent-directory `wp-config.php`.

`--skip-check` flag (2.5.1) defers `DB_NAME`/`DB_USER` validation — useful for CI bootstrap before DB is provisioned. SQLite-aware: `--dbname` and `--dbuser` optional when SQLite drop-in is detected.

## `wp core check-update-db` — read-only counterpart (core-command 2.1.24)

Safe to run in monitoring without triggering migrations. The missing read-only sibling of `wp core update-db`.

Plus in this stretch:

- `--skip-locale-check` on `wp core download` (proceeds when locale package isn't published yet — common right after `.0` releases)
- `wp core update-db --network` respects `WP_NETWORK_ADMIN_PAGE` / network ID in multinetwork installs
- `wp core update --locale=...` no longer requires `--force` for locale-only changes
- Admin password is now `wp_slash()`-safe in `wp core install`

## `wp doctor` — multisite checks + result filter (doctor-command 2.2.0 / 2.3.0)

First multisite-aware doctor checks ever shipped. New checks for site count, network options, required network plugins. `wp doctor check` gained a result filter for CI piping. `cache-flush` check returns early on first match and prints the file list. 2.3.1 fixed a false positive in `cron-duplicates` for hooks with unique args.

## `wp profile hook --search` (profile-command 2.1.6)

Filter hook results by callback name — closes the long-standing "wall of output" UX. URL-driven frontend vs. backend selection: `wp profile hook ... <url>` dispatches the right code path automatically.

## `wp i18n audit` — schema-driven audit with JSON + GitHub Actions output (i18n-command 2.7.0)

New auditing command surfaces translation issues with structured output. Drop straight into a GitHub Actions matrix, no parsing.

```bash
wp i18n audit --format=json
wp i18n audit --format=github-actions
```

Major i18n release also adds:

- `wp i18n make-php --pretty-print`
- `wp i18n update-po --purge` controls removal of obsolete translations
- **`wp i18n make-json` purge flags removed** — PO files are now source of truth (breaks scripts relying on prior purge semantics)
- PO files preserve POT order on update; `PO-Revision-Date` auto-updates
- File-header extraction records line numbers
- Translatable strings extracted from Blade component prop bindings
- `theme-i18n.json` schema gained `border.radiusSizes`
- 2.7.1 requires WP-CLI 2.13 (composer warning)

## `wp dist-archive` — PHP `ZipArchive` replaces external `zip` (dist-archive-command 3.2.0)

Works in containers without the `zip` binary (Alpine, distroless). Performance: `.distignore` directories are pruned at descent rather than filtered after walking — large `node_modules` no longer walked. 3.1.0 lowered PHP requirement to 7.2.

## `wp package is-installed`, `get`, targeted update (package-command 2.7.0)

Fills automation gaps. Default `wp package install` version flipped from "default branch" to `@stable` — meaningful change for anyone scripting installs.

```bash
wp package is-installed wp-cli/profile-command   # exit 0/1 — for shell guards
wp package get wp-cli/profile-command            # single-package details
wp package update wp-cli/profile-command         # was global-only
wp package list --skip-update-check              # avoid network in CI
wp package install vendor/pkg:1.2.3              # version suffix on git-URL installs
```

`--no-interaction` prevents Git/SSH credential prompts. Auto-appends `.git` for GitHub/GitLab URLs missing it. Index repository marked non-canonical so Packagist resolves newer versions. 2.6.1 switched to Packagist V2 Metadata API (faster lookups).

## `wp plugin install` matures (extension-command 2.3.0)

- `wp plugin install --slug=<custom>` — control install directory (for vendored or licensed plugins where filename matters)
- `wp plugin install-dependencies` — installs WP 6.5 `Requires Plugins` header dependencies
- `--with-dependencies` on `wp plugin install` — installs declared deps automatically
- `wp plugin update --all` now skips VCS-controlled directories by default — protects against `git` stomping
- `wp plugin check-update` and `wp theme check-update` (read-only sibling of update)

## New package commands shipped in the window

- `wp media replace` — replace an attachment file in place (preserve ID, URLs, references)
- `wp media prune` — delete attachments not referenced anywhere
- `wp post revision restore`, `diff`, `prune`
- `wp font *` — Font Library subcommands
- `wp user privacy-request *` — handle GDPR exporter/eraser requests
- `wp menu item add-post-type-archive`, `wp menu item get`
- `wp term prune` — delete unused terms
- `wp site get` — retrieve single-site details (was list-only)
- `wp sidebar get`, `wp sidebar exists`
- `wp widget patch` — atomic widget setting updates
- `wp cache pluck`, `wp cache patch`, `wp transient pluck`, `wp transient patch` — atomic nested-array surgery on cached blobs (mirrors the `wp option pluck/patch` family)

## Behavior changes worth knowing

- `wp post delete` no longer requires `--force` for already-trashed posts (deletes them outright)
- `wp site empty` — O(n) cache invalidation collapsed to a single `wp_cache_flush()` (perf win on big sites)
- `wp scaffold block` is still deprecated (use `@wordpress/create-block`)

## Sources

- [WP-CLI 2.12.0 release](https://github.com/wp-cli/wp-cli/releases/tag/v2.12.0)
- [WP-CLI 2.12.0 dev note](https://make.wordpress.org/cli/2025/05/07/wp-cli-v2-12-0-release-notes/)
- [config-command 2.5.0](https://github.com/wp-cli/config-command/releases)
- [core-command 2.1.24](https://github.com/wp-cli/core-command/releases)
- [doctor-command 2.2.0 / 2.3.0](https://github.com/wp-cli/doctor-command/releases)
- [profile-command 2.1.6 / 2.1.7](https://github.com/wp-cli/profile-command/releases)
- [i18n-command 2.7.0](https://github.com/wp-cli/i18n-command/releases)
- [dist-archive-command 3.2.0](https://github.com/wp-cli/dist-archive-command/releases)
- [package-command 2.7.0](https://github.com/wp-cli/package-command/releases)
- [extension-command 2.3.0](https://github.com/wp-cli/extension-command/releases)
