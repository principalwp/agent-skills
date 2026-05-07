# Buried subcommands worth knowing

Subcommands that don't show up in the tutorial roundup but solve common problems cleanly.

## `wp option pluck` / `patch`

Read or write a nested value inside an option without round-tripping through PHP:

```bash
# Read settings.featured.title from option 'my_plugin_settings'
wp option pluck my_plugin_settings settings featured title

# Set settings.featured.title without touching the rest of the array
wp option patch update my_plugin_settings settings featured title 'New Title'

# Delete a nested key
wp option patch delete my_plugin_settings settings featured title

# Insert into an array
wp option patch insert my_plugin_settings settings featured 'New Item'
```

Replaces a lot of ad-hoc `wp eval` scripts. Same family exists for post meta:

```bash
wp post meta pluck 42 my_complex_meta nested key
wp post meta patch update 42 my_complex_meta nested key 'value'
```

## `wp post meta clean-duplicates`

Removes duplicate post meta rows in-place:

```bash
wp post meta clean-duplicates 42
```

A documented data-hygiene win after botched migrations or plugins that called `add_post_meta` (instead of `update_post_meta`) in a loop.

## `wp option get-autoload` / `set-autoload`

Inspect and mutate the `autoload` flag on individual options without raw SQL:

```bash
# Find autoloaded options larger than 100KB
wp option list --autoload=yes --format=count
wp option list --autoload=yes --orderby=size_bytes --order=DESC --fields=option_name,size_bytes

# Stop autoloading a specific bloated option
wp option set-autoload my_huge_option no
```

Autoload bloat is one of the top WordPress performance problems. Using these subcommands keeps you out of `$wpdb->update` and any chance of malformed SQL.

## `wp transient` family

```bash
# Where are transients stored?
wp transient type
# → external_object_cache  (Redis/Memcached drop-in is wired up)
# → database               (drop-in not loaded — investigate)

# Clean up only expired transients
wp transient delete --expired

# Nuclear: delete everything (cache stampede risk on busy sites)
wp transient delete --all
```

## `wp core verify-checksums` / `wp plugin verify-checksums`

Compare core/plugin files against WordPress.org MD5 checksums — lightweight intrusion detection and partial-install corruption detection:

```bash
wp core verify-checksums                             # core
wp plugin verify-checksums --all                     # every plugin from .org
wp plugin verify-checksums --all --strict             # also flag readme.txt drift
wp plugin verify-checksums --all --include-root       # flag rogue files at WP root
wp plugin verify-checksums --all --exclude=foo.txt    # ignore known-good drift
```

Run as a CI gate after deployments. Plugins not hosted on WordPress.org are skipped automatically.

## `wp profile` (separate package)

Not bundled — install once:

```bash
wp package install wp-cli/profile-command
```

Then:

```bash
wp profile stage                          # break a request into bootstrap/main_query/template
wp profile stage bootstrap --spotlight    # focus on bootstrap stage, drop the noise
wp profile hook --all --spotlight         # which hooks/callbacks are slow
wp profile eval '$x = expensive_thing();' # profile arbitrary PHP
wp profile eval-file my-script.php
```

Replaces the painful "install Query Monitor on a slow site" loop. Pairs with `wp doctor`.

## `wp doctor` (separate package)

```bash
wp package install wp-cli/doctor-command
wp doctor list
wp doctor check --all                     # run every check
wp doctor check autoload-options-size cron-count  # specific checks
```

Configurable diagnostic checks (autoload size, cron count, plugin/theme version drift, etc.). Custom checks are loadable via YAML config. Useful in CI as a gate.

## `wp media regenerate --image_size=`

After changing image size definitions in `theme.json` or `add_image_size()`:

```bash
# Only regenerate the new size, only for posts missing it (much faster)
wp media regenerate --image_size=medium-square --only-missing
```

Without `--image_size`, this regenerates **every** size for every attachment — hours of needless work on a large library.

## `wp scaffold`

Generates boilerplate for new code:

```bash
wp scaffold plugin my-plugin
wp scaffold child-theme my-child --parent_theme=twentytwentyfour
wp scaffold post-type book
wp scaffold taxonomy genre --post_types=book
wp scaffold block my-block --plugin=my-plugin
wp scaffold plugin-tests my-plugin
```

Worth using as a starting point even if you'll modify heavily — sets up the full file structure and metadata correctly.

## `wp cli` self-management

```bash
wp cli has-command "post meta pluck"      # exits 0/1; use in shell scripts that should degrade gracefully
wp cli cmd-dump --format=json             # entire command tree as JSON (codegen, docs)
wp cli param-dump --format=json           # all global params with descriptions
wp cli cache clear                        # resolves "stale package" weirdness after package upgrades
wp cli completions --line="wp post "      # raw completions data (used by tab-completion shell hook)
```
