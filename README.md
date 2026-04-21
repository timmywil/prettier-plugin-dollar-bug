# @trivago/prettier-plugin-sort-imports dollar sign bug

Minimal reproduction of a bug where `@trivago/prettier-plugin-sort-imports` corrupts `.svelte` files containing dollar signs in string literals, causing a `prettier-plugin-svelte` parse error.

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
2. At least one other plugin that registers a `svelte` parser alongside `prettier-plugin-svelte` (e.g. `prettier-plugin-tailwindcss`)

This repo uses `prettier-plugin-tailwindcss` as the other plugin. Removing it makes the bug disappear because `@trivago/prettier-plugin-sort-imports`'s svelte preprocessor is no longer triggered through the parser wrapping chain.

## Root cause

In `@trivago/prettier-plugin-sort-imports`'s [svelte-preprocessor.js](https://github.com/trivago/prettier-plugin-sort-imports/blob/master/src/preprocessors/svelte-preprocessor.ts), the `sortImports` function extracts script content, processes it, and reinserts it using `String.prototype.replace`:

```js
const result = code.replace(snippet, `\n${preprocessed}\n`);
```

In JavaScript, the second argument to `String.prototype.replace` treats `$` as a special character. When the script contains `'$'` (a single-quoted dollar sign, which `singleQuote: true` produces), the `$'` in the replacement string is interpreted as "insert the portion of the string **after** the match". This injects the entire template into the middle of the script:

```js
// What the code expects:
"<script>\n  return '$' + n.toFixed(2);\n</script>\n\n<p>hello</p>"

// What actually happens — $' expands to everything after </script>:
"<script>\n  return '</script>\n\n<p>hello</p> + n.toFixed(2);\n</script>\n\n<p>hello</p>"
```

The corrupted script content is then passed to `prettier-plugin-svelte`'s `snipScriptAndStyleTagContent`, which base64-encodes the corruption. Downstream parsers receive text with a duplicate `</script>` tag, causing the `element_invalid_closing_tag` error.

## Fix

Use a function replacement to avoid `$` interpolation:

```js
const result = code.replace(snippet, () => `\n${preprocessed}\n`);
```
