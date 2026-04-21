# prettier-plugin-svelte dollar sign bug

Minimal reproduction of a bug where `prettier-plugin-svelte` errors on `.svelte` files containing dollar signs in string literals.

## Reproduce

```sh
npm install
npx prettier --write src/App.svelte
```

This produces:

```
[error] src/App.svelte: element_invalid_closing_tag: `</script>` attempted to close an element that was not open
```

## Requirements

Two conditions are required:

1. `singleQuote: true` in prettier config
2. At least two other plugins that register their own `svelte` parser alongside `prettier-plugin-svelte`

This repo uses `@trivago/prettier-plugin-sort-imports` and `prettier-plugin-tailwindcss`, which both register a `svelte` parser. Replacing either with a plugin that doesn't makes the bug disappear.

## Why prettier-plugin-svelte?

When multiple plugins register a `svelte` parser, they form a wrapping chain — each plugin's parser calls the next. `prettier-plugin-tailwindcss` is designed to run last, so the actual call chain is: tailwindcss wraps sort-imports, which wraps prettier-plugin-svelte's parser. This multi-pass wrapping is what exposes the bug — without it, prettier-plugin-svelte's parser runs alone and the placeholder logic works fine.

The `✂prettier:content✂` placeholder mechanism is owned by `prettier-plugin-svelte`. It replaces `<script>` content with base64 placeholders before other plugins run, then restores it afterward. When we intercept the text passed between parsers at runtime, we can see the corruption happens in this placeholder layer — the script body leaks outside the placeholder:

```
<script ✂prettier:content✂="CgogI...">{}</script>

<p>{`$${price.toFixed(2)}`}</p> + n.toFixed(2);
  }

</script>                          <-- duplicate, causes the error

<p>{`$${price.toFixed(2)}`}</p>
```

The other two plugins receive this already-corrupted text as input to their parsers. They don't produce it — the corruption is present before they run.

The `'$'` (single-quoted dollar sign, produced when `singleQuote: true` rewrites `"$"`) appears to break the placeholder extraction boundary, causing the rest of the script content to spill into the template. This is consistent with the placeholder logic living in `prettier-plugin-svelte`, not in the other plugins.
