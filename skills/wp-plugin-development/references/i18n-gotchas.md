# Internationalization gotchas

## .l10n.php preferred (6.5+)

Since WP 6.5, `.l10n.php` files are the preferred translation format — faster than `.po`/`.mo` because they leverage PHP opcache. WordPress auto-loads `.l10n.php` before falling back to `.mo`. Generate them via `wp i18n make-php`.

If your build pipeline still only emits `.mo`, you're leaving a measurable performance win on the floor — translation lookups run on every request that uses `__()`.

## Text domain "default"

The `'default'` text domain is reserved for WordPress core translations. Passing `'default'` (or omitting the domain entirely) in plugin code routes lookups through core's translation tables and silently fails to translate plugin strings even when the plugin has matching translations loaded.

## _x context parameter

`_x($text, $context, $domain)` adds disambiguation context. Example: `_x('Post', 'noun', 'my-plugin')` vs `_x('Post', 'verb', 'my-plugin')`. Translators see the context, so the same English string can map to different translations in languages where the noun and verb diverge.

If you have repeated short strings like "Post", "Read", "Free" — disambiguate them, otherwise translators have to guess and pick one rendering for all instances.

## _n parameter order

```php
_n( $single, $plural, $number, $domain );
// NOT _n( $number, $single, $plural, $domain )
```

The number comes after the strings, not before. Easy to flip if you're used to other i18n libraries that put the count first.

## _n_noop and translate_nooped_plural

Register strings for extraction without translating immediately:

```php
$messages = _n_noop( '%s item', '%s items', 'my-plugin' );
// Later, when you know $count:
echo translate_nooped_plural( $messages, $count, 'my-plugin' );
```

Use this when the strings live in static class properties or config arrays where you can't call `_n()` directly because the count isn't known yet — but you still need the strings to appear in the POT file.

## determine_locale vs get_locale

On admin screens:
- `determine_locale()` — returns the **user's** locale (per-user setting).
- `get_locale()` — returns the **site's** locale.

On the frontend they typically return the same value.

If you're sending an email to an admin and want it localized to *their* preferred language (not the site's), use `determine_locale()` — but be careful: by the time `wp_mail` runs the request locale may already be the recipient's; use `switch_to_user_locale()` for explicit per-user mail.

## JS translations

```php
wp_set_script_translations( 'my-script-handle', 'my-plugin', plugin_dir_path( __FILE__ ) . 'languages' );
```

Requires a matching `.json` translation file in Jed format. Generate with `wp i18n make-json languages/`. Skip this step and your JS strings remain untranslated even though PHP strings are fine — a common "half my plugin is translated" bug.
