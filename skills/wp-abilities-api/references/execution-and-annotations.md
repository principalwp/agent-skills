# Abilities API execution flow and annotations

## Execution pipeline (7-step)

`WP_Ability::execute()` follows a strict pipeline:

```
1. normalize_input
2. validate_input (against input_schema)
3. check_permissions (permission_callback)
4. wp_before_execute_ability (action hook)
5. do_execute (execute_callback)
6. validate_output (against output_schema)
7. wp_after_execute_ability (action hook)
```

Hooks at steps 4 and 7 allow cross-cutting concerns (logging, auditing) without modifying the ability itself.

## Ability annotations

Three annotation properties control behavior metadata:

| Annotation    | Default | Meaning                                       |
|---------------|---------|-----------------------------------------------|
| `readonly`    | `false` | Ability only reads data, never modifies       |
| `destructive` | `true`  | Ability removes or irreversibly changes data  |
| `idempotent`  | `false` | Repeated calls produce the same result        |

**The defaults are unsafe — always set all three explicitly.** A read-only ability that doesn't set `destructive => false` is treated as state-modifying by MCP clients, which routes it to confirmation flows and slows the agent loop. Defaults are intentionally cautious so missing annotations fail closed, but that means "I forgot to annotate" stays silent until production.

```php
'meta' => [
    'annotations' => [
        'readonly'    => true,   // never default this
        'destructive' => false,  // never default this
        'idempotent'  => true,   // never default this
    ],
],
```

## HTTP method mapping (REST)

When `show_in_rest = true`, annotations determine the HTTP method:

| Condition | HTTP method |
|-----------|-------------|
| `readonly = true` | `GET` |
| `destructive = true` AND `idempotent = true` | `DELETE` |
| Everything else | `POST` |

## REST exposure

Opt-in only — default is `false`:

```php
wp_register_ability( 'my-plugin/action', [
    // ...
    'meta' => [ 'show_in_rest' => true ], // Required for REST visibility
] );
```
