# 09 · Strict Mode Best Practices

`"strict": true` is actually a bundle of eight separate flags, and
there are several more useful checks that aren't included in `strict`
at all and must be opted into individually. This module inspects what
each flag actually catches, and demonstrates a real gap that `strict`
alone doesn't close.

## What `strict: true` turns on

```json
{
  "compilerOptions": {
    "strict": true
  }
}
```

is shorthand for:

```json
{
  "compilerOptions": {
    "noImplicitAny": true,
    "strictNullChecks": true,
    "strictFunctionTypes": true,
    "strictBindCallApply": true,
    "strictPropertyInitialization": true,
    "noImplicitThis": true,
    "useUnknownInCatchVariables": true,
    "alwaysStrict": true
  }
}
```

## `strictNullChecks` — the most impactful single flag

```typescript
function findUser(id: number): { name: string } | undefined {
  return id === 1 ? { name: "Ada" } : undefined;
}
const user = findUser(1);
if (user) {
  console.log(user.name);
}
```

```text
Ada
```

Without `strictNullChecks`, `undefined` is silently assignable to
everything, and `user.name` would compile even without the `if (user)`
guard — crashing at runtime for any id other than `1`. With it,
`user.name` directly (no guard) is a compile error, forcing exactly
the check shown above before access.

## `strictPropertyInitialization` — catches uninitialized class fields

```typescript
class Service {
  name: string;
  constructor(name: string) {
    this.name = name;
  }
}
console.log(new Service("api").name);
```

```text
api
```

If the constructor forgot to assign `this.name`, `strictPropertyInitialization`
flags `name: string;` itself as an error ("Property 'name' has no
initializer and is not definitely assigned in the constructor") —
without this flag, a forgotten assignment compiles fine and `name` is
silently `undefined` at runtime despite its declared type of `string`.

## The gap: `noUncheckedIndexedAccess` is not part of `strict`

```typescript
const scores: Record<string, number> = { ada: 100 };
const score = scores["grace"];
console.log(score.toFixed(0));
```

Compiled and run with `--strict` alone:

```bash
tsc j.ts --strict
node j.js
```

```text
TypeError: Cannot read properties of undefined (reading 'toFixed')
```

**This compiles cleanly under full `--strict`.** `Record<string, number>`'s
index signature claims every string key maps to a `number` — but
`"grace"` was never actually added to `scores`, so `scores["grace"]` is
really `undefined` at runtime, and `score.toFixed(0)` crashes. `strict`
alone does not catch this, because index signatures are treated as
"always present" unless told otherwise.

Adding the one extra flag:

```bash
tsc --noEmit --strict --noUncheckedIndexedAccess j.ts
```

```text
error TS18048: 'score' is possibly 'undefined'.
```

Same code, now a **compile-time** error instead of a runtime crash.
`noUncheckedIndexedAccess` types every index-signature and array access
as `T | undefined`, forcing a check before use — it's not part of
`strict` because it was added years later and would break too much
existing "strict-clean" code by default, but most new projects should
turn it on explicitly.

## Other flags worth enabling beyond `strict`

```json
{
  "compilerOptions": {
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "noImplicitOverride": true,
    "exactOptionalPropertyTypes": true,
    "noFallthroughCasesInSwitch": true
  }
}
```

`noImplicitOverride` requires an explicit `override` keyword when a
subclass method overrides a base class method — without it, renaming a
base class method silently orphans the "override" in the subclass
instead of flagging it. `exactOptionalPropertyTypes` distinguishes
`{ x?: string }` (key can be absent) from `{ x: string | undefined }`
(key present, value `undefined`) — a real distinction that plain
optional properties blur by default.

## Traps

**Turning on `strict` for the first time on an existing large codebase
produces an overwhelming number of errors at once.** Most real
migrations enable the flags individually (`noImplicitAny` first,
usually, then `strictNullChecks`) rather than flipping `strict: true`
all at once, fixing each flag's errors before moving to the next.

**`strict: true` in `tsconfig.json` doesn't retroactively check
`node_modules`'s own `.d.ts` files against the same standard** — that's
what `skipLibCheck` controls separately, and most projects want
`skipLibCheck: true` *alongside* `strict: true`, not instead of it.

**A flag being off doesn't mean the underlying bug can't happen — it
means the compiler won't catch it for you.** The `scores["grace"]`
crash above happens at runtime regardless of any compiler flag; the
flags only change whether TypeScript catches it before you ship.

**Adding `noUncheckedIndexedAccess` to an existing project surfaces
many new `possibly undefined` errors on code that "worked" before** —
often on array access (`arr[i]`) throughout the codebase — because it
applies to every indexed access, not just object index signatures.

## Cheat sheet

| Flag | In `strict`? | Catches |
|---|---|---|
| `noImplicitAny` | Yes | Untyped parameters/variables defaulting to `any` |
| `strictNullChecks` | Yes | Using a possibly-`null`/`undefined` value without a check |
| `strictPropertyInitialization` | Yes | Class fields never assigned in the constructor |
| `strictFunctionTypes` | Yes | Unsound function parameter variance |
| `noUncheckedIndexedAccess` | **No** | Index/array access assumed always present |
| `noImplicitOverride` | **No** | Subclass method silently no longer overriding anything |
| `exactOptionalPropertyTypes` | **No** | Blurring "key absent" vs. "key present, value `undefined`" |

## Exercise

Take the `scores` example and add `noUncheckedIndexedAccess: true` to a
real `tsconfig.json`. Fix the resulting error properly — not by
suppressing it, but by adding a check (`if (score !== undefined)`) —
and confirm the fixed version both compiles under
`--noUncheckedIndexedAccess` and runs without throwing when queried for
a key (`"grace"`) that was never added.
