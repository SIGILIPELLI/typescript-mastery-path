# 08 · Third-Party Types

Most npm packages either ship their own `.d.ts` files or get typed
separately via `@types/*` (the DefinitelyTyped project) — but you'll
still regularly hit a package with no types at all, or want to extend a
type that isn't yours. This module covers writing your own declaration
files, module augmentation, and declaration merging.

## `@types` packages — the common case

```bash
npm install lodash
npm install -D @types/lodash
```

Most popular untyped JS libraries have a matching `@types/<package>`
package maintained by the community. TypeScript automatically picks
these up from `node_modules/@types` with no extra configuration —
`import _ from "lodash"` is fully typed as soon as both packages are
installed.

## Writing a declaration file for an untyped module

Say you depend on a small internal JS module with no types:

```javascript title="legacy/index.js"
exports.greet = function (name) {
  return "Hello, " + name + "!";
};
```

Without a `.d.ts` file, importing it gives every export the type `any`.
Add a matching declaration file next to it:

```typescript title="legacy/index.d.ts"
export function greet(name: string): string;
```

```typescript
import { greet } from "./legacy/index.js";
console.log(greet("Ada"));
```

```text
Hello, Ada!
```

TypeScript matches `legacy/index.d.ts` to `legacy/index.js` purely by
file name — no import or reference needed. This is exactly what
`@types/lodash` does at a larger scale for `lodash`.

## Module augmentation — adding to a type you don't own

```typescript
declare global {
  interface String {
    shout(): string;
  }
}
String.prototype.shout = function (this: string): string {
  return this.toUpperCase() + "!";
};
console.log("hello".shout());
```

```text
HELLO!
```

`declare global { ... }` is required here specifically because this
file has a top-level `import`/`export`, which makes TypeScript treat it
as a **module** — inside a module, a plain `interface String { ... }`
would only add `shout` to a *local* shadow of `String`, not the real
global one. `declare global` is the escape hatch back to global scope
from inside a module file.

## Declaration merging — the mechanism behind both of the above

```typescript
interface PluginOptions {
  name: string;
}
interface PluginOptions {
  version: string;
}
const opts: PluginOptions = { name: "logger", version: "1.0.0" };
console.log(opts);
```

```text
{ name: 'logger', version: '1.0.0' }
```

Two `interface` declarations with the same name **merge** into one
combined interface — this isn't a special case for built-ins, it's a
general TypeScript rule, and it's exactly how you'd extend a third-party
library's exported interface (e.g. adding a custom field to Express's
`Request` type) without modifying that library's source.

## Traps

**Forgetting `declare global` inside a module file makes augmentation
silently local.** `interface String { shout(): string }` at the top of
a file with an `import` statement compiles without error — it just
doesn't do what you wanted. `"hello".shout()` elsewhere in your project
still fails with "Property 'shout' does not exist," and the error
message gives no hint that the fix is `declare global`.

**An ambient module declaration can't use a relative path.**
`declare module "./legacy/index.js" { ... }` fails with
`TS2436: Ambient module declaration cannot specify relative module name`.
Ambient `declare module "name"` is for bare package specifiers
(`"lodash"`, `"my-untyped-package"`); for a local file, name the
declaration file identically to the source file instead
(`index.d.ts` next to `index.js`), as shown above.

**A `.d.ts` file with no exports and no `declare global` is a global
script, which can silently clash with other global declarations.** If
you meant to scope your types to a module, add at least one `export`
(even `export {}`) to force module mode.

**`@types/*` versions can drift from the actual package version** —
installing `lodash@5` with `@types/lodash@4` still installed gives you
types for the *old* API, so a method that was removed in v5 can still
type-check and then throw `TypeError: ... is not a function` at
runtime.

## Cheat sheet

| Situation | Fix |
|---|---|
| Untyped npm package | `npm install -D @types/<package>` if it exists |
| Untyped local `.js` file | Add a sibling `.d.ts` file with matching name |
| Untyped package, no `@types` exists | Add `declare module "package-name" { ... }` in a project-level `.d.ts` |
| Adding a method to a built-in inside a module file | `declare global { interface X { ... } }` |
| Extending a third-party exported interface | A second `interface` declaration with the same name (merges) |

## Exercise

Write a declaration file for a fictional untyped package
`"metrics-lite"` that exports a default function
`track(event: string, props?: Record<string, unknown>): void`, using
`declare module "metrics-lite" { ... }` in a standalone `.d.ts` file
(no matching `.js` needed for this exercise — just prove it type-checks
by importing and calling `track` from a `.ts` file with `tsc --noEmit`).
