# WordPress Playground release deltas (last 12 months)

Concrete deltas from `WordPress/wordpress-playground` releases v1.0.33 (2025-05-08) → v3.1.29 (2026-05-07). Sourced from GitHub releases, the Playground 2025 year-in-review, and Make/Playground posts.

## Blueprints v1 → v2 split (v3.0.0, 2025-09-19)

Major release made v2 a parallel schema, **not a replacement**. v1 types were renamed (`BlueprintDeclaration` → `BlueprintV1Declaration`). The v2 runner is gated behind `--experimental-blueprints-v2-runner` until it stabilizes.

If you ship blueprints, audit your import paths. The `@wp-playground/blueprints` package now exposes both schemas; `Blueprint` (unprefixed) still points at v1 in v3.x.

## CLI maturity — `start`, `--phpmyadmin`, `--wordpress-install-mode`, `--auto-mount`, `--xdebug`

`@wp-playground/cli` shipped most of the production-grade flags in this window:

- **`start`** command (v3.0.41, Feb 2026) — convenience wrapper that runs server + auto-mount + auto-open. Closes the gap between "just spin up a WP" and writing a blueprint.
- **`--phpmyadmin`** (v3.0.48) — bundle phpMyAdmin into the running instance.
- **`--wordpress-install-mode`** (v3.0.20) — four modes (`full`, `core-only`, `skip`, `auto`) for explicit control over what core does on first boot. Relevant for snapshot-driven workflows where you don't want the install wizard to run.
- **`--auto-mount=path`** (v2.0.12) — point at a plugin/theme folder; Playground mounts and activates.
- **`--no-auto-mount`** (v3.1.20) — actually works on `start` now (previous fix incomplete).
- **`--debug`** is deprecated in favor of `--verbosity=debug`.
- **`--define-bool`** / **`--define-number`** — type-aware constant injection into wp-config.
- **`--xdebug`** + **`--experimental-unsafe-ide-integration`** — Xdebug bridge to host IDEs (v3.0.x). VS Code Xdebug integration ships separately as the host-bridge package.
- **`--internal-cookie-store`** — isolated cookie jar for repeatable test runs.
- **`--blueprint-may-read-adjacent-files`** — needed when blueprints reference local files via `file:` resources.
- **ZIP-as-blueprint** resolver — point `--blueprint=path/to/wp.zip` to import a snapshot directly.

## `wp-now` deprecated — migrate to `@wp-playground/cli`

`@wp-now/wp-now` is officially deprecated. Anywhere in your codebase still referencing `@wp-now/wp-now` should migrate to `@wp-playground/cli` with `--auto-mount`. The replacement runtime: `wp-env --runtime=playground` (v3.0.x, Feb 2026), which collapses the Docker/Playground split that was forcing agencies to maintain parallel local-dev rails.

## PHP versions — 8.3 default, 7.2/7.3 removed

- **PHP 8.3** became default (v1.2.3).
- **PHP 7.2 and 7.3 removed** (v3.0.43). If your blueprint pinned an old PHP for legacy plugin testing, this is a hard break.
- New extensions in the window: SOAP, ImageMagick, AVIF, Redis, memcached, intl Asyncify.
- CLI SAPI now in the web build; `php.cli()` callable from browser (v2.0.16) — opens browser-side WP-CLI invocation.

## Mount semantics — Posix paths, symlinks, multi-worker

- Single-file NODEFS mounts became first-class (v1.2.3).
- Multi-worker `/wordpress` mounts via Asyncify (v1.2.2).
- Posix path conversion for symlinks (v3.0.48) — Windows hosts now mount symlinked plugins correctly.
- Parent-dir mount for symlinked-file `__DIR__` (v3.1.x) — fixes plugins that walk `__DIR__` to load includes.
- Symlinked directory support (v3.1.29-area).

## New blueprint steps and changed step semantics

- `importWxr` switched to the canonical `wordpress-importer` plugin under the hood (v3.0.0). Behavior is now identical to the WP admin importer; pre-3.0 quirks are gone.
- `enableMultisite` defines `$_SERVER['HTTP_HOST']` (v3.0.54) — fixes a class of multisite blueprint failures where the boot hit `null` host.
- New resource type `git:Directory` (v3.0.13) — clone an actual `.git` checkout instead of a tarball. Useful for branch-aware fixtures.

## Performance — OPCache, multi-worker, instance cap, lower memory

- OPCache enabled by default (v2.0.0) — drove the 42% response-time drop reported in the year-in-review.
- Multi-worker browser execution (v1.2.x).
- PHP-instance cap at 2 (v3.0.47) — protects against runaway in-browser instantiation.
- Lower initial PHP memory (v3.0.47) — better behavior on memory-constrained tabs.

## OPFS persistence productized via Personal Playground

`my.wordpress.net` shipped as the productized Personal Playground (v3.0.46 → v3.1.x), with multi-tab coordination, health-check recovery, app catalog, backup UI, and an explicit OPFS flush API. If you build embedded Playground experiences, the same persistence primitives are available via the package — no need to roll your own coordination layer.

## Studio integration

- **Studio 1.6.0** (2025-10-08) added Blueprints to the new-site flow — a Blueprint URL or local file becomes a fixture for new sites.
- **Studio 1.7.0** (2026-01-27) shipped a `studio` CLI with `site create --blueprint` — ready for CI use.

## Host bridges — agent-skills, MCP, Xdebug, Devtools

- **agent-skills** package (April 2026) — first-party Claude Code / Gemini CLI / Copilot / Cursor / Codex bridges. The blueprint agent skill codifies the LLM mistakes worth avoiding (`pluginZipFile` vs `pluginData`, missing `wp-load.php` in `runPHP`, `git:directory` `refType` rules, mu-plugin gotchas).
- **MCP server for browser Playground** — agents can drive a running Playground instance via MCP.
- **CORS proxy** for embeds (v2.0.0); `wasm.wordpress.net` accepted as official origin.
- **Devtools extension** v0.0.1 (2025-12-17) — browser devtools panel for Playground state.
- **VS Code Xdebug bridge** — IDE step-debugging into Playground PHP.

## WP version cadence + version-handling fixes

Playground deploys after every WP major and beta (issue #2378). Several version-string handling bugs got fixed across v3.0.19, v3.0.22, v3.1.x:

- `.0` versions (e.g. `6.9.0`) now resolve correctly.
- `null` / `latest` / `beta` slugs accepted.
- Two-part version strings (e.g. `6.9`) accepted.

## Live Blueprint Editor (Dec 2025)

Hosted editor at `playground.wordpress.net/blueprint-builder/` — design and run blueprints interactively with hot reload. Useful for developing blueprints before committing them to a repo. Linked from the blueprint agent skill.

## Things that are *not* there (worth knowing)

- **"Surf"** — no signal in the changelog or `make.wordpress.org/playground` for the last 12 months. If something with that name exists in the WP Playground orbit, it's not a release artifact in this window.
- **Native streaming / SSE** inside Playground PHP — still proxied via host bridge for HTTP-style transports.

## Sources

- [WordPress/wordpress-playground releases](https://github.com/WordPress/wordpress-playground/releases)
- [WordPress/wordpress-playground CHANGELOG](https://github.com/WordPress/wordpress-playground/blob/trunk/CHANGELOG.md)
- [Playground 2025 year-in-review](https://make.wordpress.org/playground/2026/01/15/2025-in-review/)
- [Live Blueprint Editor](https://make.wordpress.org/playground/2025/12/17/live-blueprint-editor/)
- [Playground agent skills (April 2026)](https://make.wordpress.org/playground/2026/04/15/agent-skills/)
- [Studio 1.6.0 release](https://wordpress.com/blog/2025/10/08/studio-1-6/)
- [Studio 1.7.0 CLI](https://wordpress.com/blog/2026/01/27/studio-1-7/)
