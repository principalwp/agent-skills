# `wp eval`, `wp eval-file`, and `wp shell`

WP-CLI is also a PHP script runner with WordPress loaded. Three commands matter:

| Command | Use when |
|---------|----------|
| `wp eval '<php>'` | Inline one-liner |
| `wp eval-file <path>` | Multi-line script in a file (or `-` for stdin) |
| `wp shell` | Interactive REPL (Boris/PsySH if installed) |

## `wp eval` requires `global $wpdb`

Eval runs your code inside a method body, so `$wpdb` (and other WP globals) are not in scope by default:

```bash
# WRONG — silently runs against $wpdb = null
wp eval '$wpdb->query( "DELETE FROM wp_options WHERE option_name LIKE \"_transient_%\" " );'

# RIGHT
wp eval 'global $wpdb; $wpdb->query( "DELETE FROM wp_options WHERE option_name LIKE \"_transient_%\" " );'
```

LSP/IntelliSense completes `$wpdb` happily; the error is runtime-only. Same applies to `eval-file` — the loaded file runs as a method body.

## `wp eval-file` reads from stdin with `-`

For multi-line PHP without writing a temp file:

```bash
wp eval-file - <<'PHP'
global $wpdb;
$expired = $wpdb->query(
    "DELETE FROM {$wpdb->options}
     WHERE option_name LIKE '_transient_timeout_%'
       AND option_value < UNIX_TIMESTAMP()"
);
WP_CLI::log( "Deleted {$expired} expired transients." );
PHP
```

## `--skip-wordpress` when you don't need WP

```bash
wp eval-file --skip-wordpress my-data-script.php
```

Skips the WordPress bootstrap (saves ~3 seconds plus database/option-cache work). Useful when the script is pure PHP that just needs to access the file system or process input — no `wpdb`, no functions like `get_option()`.

## `--use-include` vs default eval

By default `wp eval-file` runs the file via `eval()`, which means:

- function declarations re-execute every call (fatal "function already declared")
- `global` keyword behaves oddly inside the script

Pass `--use-include` to load the file via `include` instead — function declarations work, scope behaves like a normal PHP file:

```bash
wp eval-file --use-include my-script-with-functions.php
```

## `display_errors` is overridden — fatals are silently swallowed

WP-CLI overrides PHP's `display_errors` ini setting in its bootstrap. A `Call to undefined function ...` in your `eval-file` script can print earlier output and exit `0` with the error suppressed.

To debug a script that "just exits silently":

```bash
WP_CLI_PHP_ARGS='-d display_errors=stderr' wp eval-file my-script.php
```

Or, inside the script, register a shutdown handler that re-emits the last error:

```php
register_shutdown_function( function() {
    $err = error_get_last();
    if ( $err && in_array( $err['type'], [ E_ERROR, E_PARSE, E_CORE_ERROR, E_COMPILE_ERROR ], true ) ) {
        WP_CLI::error( "Fatal: {$err['message']} in {$err['file']}:{$err['line']}" );
    }
} );
```

## `wp shell` — interactive REPL

Drops you into a PHP REPL with WordPress fully bootstrapped. Type `exit` to leave; type `restart` to reload after editing code on disk.

```
$ wp shell
wp> $post = get_post( 42 );
wp> echo $post->post_title;
Hello world
wp> global $wpdb;
wp> $wpdb->get_var( "SELECT COUNT(*) FROM {$wpdb->posts} WHERE post_status='publish'" );
=> '127'
wp> restart   # reload after editing a plugin file
```

If `boris/boris` or `psy/psysh` is installed (in WP-CLI's package autoload or the project's vendor), `wp shell` upgrades to a full REPL with history, tab completion, and pretty-printing. Without them you get a minimal `eval`-based prompt.

`wp shell --basic` forces the minimal prompt even when PsySH is available — useful in CI logs where REPL escape codes are noise.

## Composing commands inside custom commands: `WP_CLI::runcommand()`

Inside a custom command (registered via `WP_CLI::add_command`), don't shell out to `wp` again — call it in-process:

```php
$option = WP_CLI::runcommand(
    'option get my_plugin_settings --format=json',
    [ 'return' => true, 'parse' => 'json' ]
);
// $option is the decoded array.
```

Saves a full PHP startup per call and gives you typed return values. For multisite iteration where state must reset between sites, use `WP_CLI::launch_self()` instead — that does fork a subprocess, by design.

## Setting array post meta via `--format=json`

```bash
# WRONG — writes the literal serialized string into meta
wp post meta add 123 my_key 'a:1:{i:0;i:5;}'

# RIGHT
wp post meta update 123 my_key '[5,6]' --format=json
```

`wp post meta update` doubles as `add` if the key doesn't exist — prefer it over `add` for idempotency.
