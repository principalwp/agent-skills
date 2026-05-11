# Aliases and remote execution

Use this file when running WP-CLI against multiple environments, containers, or remote hosts.

## Aliases via `~/.wp-cli/config.yml`

Define per-site aliases once, refer to them anywhere:

```yaml
@dev:
  path: /var/www/dev/wp
  url: https://dev.example.com
  user: admin

@staging:
  ssh: deploy@staging.example.com:/var/www/wp
  url: https://staging.example.com

@prod:
  ssh: deploy@prod.example.com:/var/www/wp
  url: https://www.example.com

@all:
  - @dev
  - @staging
  - @prod
```

Then:

```bash
wp @prod plugin list                    # plugin list on production
wp @all core update --dry-run          # fan-out across every alias in @all
wp @prod cron event list                # remote cron inspection
```

`@all` and any group alias (a list of other aliases) runs the command sequentially against each member. Single command, fleet effect.

## Manage aliases programmatically

```bash
wp cli alias list
wp cli alias get @prod
wp cli alias add @qa --set-ssh=deploy@qa.example.com:/var/www/wp --set-url=https://qa.example.com
wp cli alias is-group @all       # is-group exits non-zero for non-group aliases
```

`wp cli has-command "post meta pluck"` is the documented way to write portable shell scripts that gracefully degrade when a package isn't installed.

## `--ssh=` accepts multiple schemes

The `--ssh=` global flag is not just SSH:

```bash
wp --ssh=ssh:user@host:/var/www/wp plugin list
wp --ssh=docker:wordpress plugin list
wp --ssh=docker-compose:web plugin list
wp --ssh=vagrant:default plugin list
```

`docker:` and `docker-compose:` execute the command inside the named container/service. Lets you run `wp` from the host instead of `docker exec`-ing into the container.

### Caveats

- The official `wp-cli` Docker image has **no SSH client**, so `--ssh=` to a remote host fails from inside that image. Build a custom image if you need SSH-from-container.
- Container `--ssh` requires WP-CLI installed inside the target container.

### Env-var escape hatches for CI

When TTY allocation breaks (most CI runners have no TTY), set:

```bash
WP_CLI_DOCKER_NO_TTY=1
WP_CLI_DOCKER_NO_INTERACTIVE=1
```

For SSH-pre-commands (e.g., `cd /var/www/site && nvm use 18 && ...`):

```bash
WP_CLI_SSH_PRE_CMD="cd /var/www/site"
```

For non-default SSH binary:

```bash
WP_CLI_SSH_BINARY=/usr/bin/ssh.custom
```

## Non-interactive SSH skips your `.bashrc`

WP-CLI proxies through `ssh`, which does **not** load shell rc files for non-interactive sessions. This means:

- `$PATH` extensions in `.bashrc` don't apply on the remote side
- `alias wp=/path/to/wp` in `.bashrc` is ignored
- Tooling installed via Homebrew, asdf, nvm, etc. that relies on shell init won't be active

This is the single most common "works locally, fails over `--ssh`" failure mode. Symptoms:

```
bash: wp: command not found
```

…even though `wp --info` works fine when you SSH in interactively.

Fixes (pick one):

1. Place `wp` (or a symlink) in a path that ssh inherits: `/usr/local/bin/wp` is reliable across distros.
2. In your remote `.bashrc`, add at the very top (before any "if not interactive" early return):
   ```bash
   shopt -s expand_aliases
   alias wp=/path/to/wp
   ```
3. Use `WP_CLI_SSH_PRE_CMD` to source what you need:
   ```bash
   WP_CLI_SSH_PRE_CMD='source $HOME/.profile && '
   ```

## `--prompt` for sensitive args

`--prompt=<arg-name>` interactively asks for a single argument instead of failing on missing input — and keeps the value out of shell history.

```bash
wp config create --dbname=wp --dbuser=wp --prompt=dbpass
```

The flag also has a no-arg form: `--prompt` alone walks every missing argument.

Useful for credentials, deploy tokens, and one-off destructive operations where you want a manual confirmation gate.

## `WP_CLI_STRICT_ARGS_MODE=1`

When a custom command has a flag with the same name as a WP-CLI global (e.g., a custom `--user` flag for "act as this user" semantics), the global parser eats the flag before the command sees it.

Set this env var to require globals to come **before** the command name and command-flags after:

```bash
# Without strict mode — --user is ambiguous
wp my-plugin sync-orders --user=42

# With strict mode — clear: globals before, command flags after
WP_CLI_STRICT_ARGS_MODE=1 wp --user=admin my-plugin sync-orders --user=42
```
