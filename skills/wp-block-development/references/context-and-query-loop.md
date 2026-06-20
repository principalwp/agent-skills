# Block context and Query Loop

Use this file when your block reads per-post data (content, title, meta, author, excerpt, etc.) and may render inside a Query Loop, archive template, or any context where multiple post instances appear on one page.

## Block context API

Blocks can receive context from ancestor blocks via `usesContext` in `block.json`. The Query Loop block (`core/post-template`) provides `postId` and `postType` to all descendant blocks.

### Declaring context consumption

In `block.json`:

```json
{
  "usesContext": [ "postId", "postType" ]
}
```

Only declare context keys you actually use. Declaring unused keys creates confusion for maintainers.

## Editor component (`edit.js`)

When a block declares `usesContext`, it receives a `context` prop containing the values provided by ancestor blocks.

### Required pattern: context-aware data fetching

```js
export default function Edit( { context } ) {
    const blockProps = useBlockProps();
    const { postId, postType } = context;

    const content = useSelect(
        ( select ) => {
            if ( postId ) {
                // Inside a Query Loop — read from the entity store.
                const post = select( 'core' ).getEditedEntityRecord(
                    'postType',
                    postType || 'post',
                    postId
                );
                return post?.content?.raw || post?.content || '';
            }
            // Standalone usage — read from the current post editor.
            return select( 'core/editor' ).getEditedPostContent();
        },
        [ postId, postType ]
    );

    // ... use content
}
```

**Key points:**

- `context.postId` is `undefined` when the block is used standalone (not inside a Query Loop). Use this to branch between the two modes.
- `select( 'core' ).getEditedEntityRecord()` fetches from the entity store which is keyed per post. This returns the correct post's data even when multiple instances render on the same page.
- `select( 'core/editor' ).getEditedPostContent()` always returns the content of the post/page/template **being edited**, not the iterated post. Using this inside a Query Loop is a bug — every block instance shows the same (wrong) data.

### The antipattern

```js
// DO NOT do this in a block that declares usesContext.
const content = useSelect(
    ( select ) => select( 'core/editor' ).getEditedPostContent(),
    []
);
```

This ignores context entirely. In a Query Loop with 10 posts, all 10 block instances show the reading time / excerpt / meta of the template being edited.

The same branching pattern applies to any post attribute (meta, title, excerpt): check `postId` first, fall back to `core/editor` for standalone usage.

## Server-side rendering (`render.php`)

The `$block->context` array contains the same values. Always prefer it over global functions, with a fallback for standalone usage:

```php
$post_id = (int) ( $block->context['postId'] ?? get_the_ID() );
if ( ! $post_id ) {
    return '';
}

$post_type = $block->context['postType'] ?? get_post_type( $post_id );
```

**Cast `$post_id` to `int` explicitly.** Context values arrive as mixed types; functions like `get_post_field()` are tolerant, but explicit casting is defensive coding that prevents subtle bugs.

## Verification

When a block reads per-post data, always test both scenarios:

1. **Standalone**: Insert the block in a regular post editor. It should display data from the post being edited.
2. **Query Loop**: Create a template or pattern with a Query Loop containing the block. Each block instance should display data from its respective post, not from the template.

## Common blocks that need this pattern

Any block that derives its output from post content, post meta, or post attributes:

- Reading time / word count
- Author bio
- Share counts
- Custom field display
- Excerpt derivatives
- Related post indicators
- Paywall / membership status

If your block falls into this category and does not use `usesContext`, it will only work correctly in the single-post editor.

## References

- Block context: https://developer.wordpress.org/block-editor/reference-guides/block-api/block-context/
- Query Loop: https://developer.wordpress.org/block-editor/reference-guides/core-blocks/#query-loop
- Entity records: https://developer.wordpress.org/block-editor/reference-guides/data/data-core/#getEditedEntityRecord
