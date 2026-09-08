# 05 · Security Considerations

TypeScript's type system doesn't stop SQL injection or XSS by itself —
but it can make certain classes of mistake structurally impossible to
compile, using a technique called **branded types**. This module builds
that, plus safer environment-variable loading, two of the more
practical security-adjacent patterns TypeScript specifically enables.

## Branded types: making unsanitized strings uncompilable where sanitized ones are required

```typescript
type Brand<T, B extends string> = T & { readonly __brand: B };

type RawInput = string;
type SafeHtml = Brand<string, "SafeHtml">;

function escapeHtml(input: RawInput): SafeHtml {
  const escaped = input
    .replace(/&/g, "&amp;")
    .replace(/</g, "&lt;")
    .replace(/>/g, "&gt;")
    .replace(/"/g, "&quot;");
  return escaped as SafeHtml;
}

function renderToPage(html: SafeHtml): string {
  return `<div>${html}</div>`;
}

const userComment = "<script>alert('xss')</script>";
const safe = escapeHtml(userComment);
console.log(renderToPage(safe));
```

```text
<div>&lt;script&gt;alert('xss')&lt;/script&gt;</div>
```

`SafeHtml` and `RawInput` are both `string` at runtime — the brand
(`{ readonly __brand: "SafeHtml" }`) exists purely in the type system
and costs nothing at runtime. But because a plain `string` isn't
assignable to `SafeHtml`, calling `renderToPage` directly with raw,
unescaped input is a **compile error**:

```typescript
renderToPage(userComment);
```

```text
error TS2345: Argument of type 'string' is not assignable to
parameter of type 'SafeHtml'.
  Type 'string' is not assignable to type '{ readonly __brand: "SafeHtml"; }'.
```

The only way to produce a `SafeHtml` value is by calling `escapeHtml` —
so as long as `escapeHtml` is correct, every call site that needs safe
HTML is forced through it. This same pattern generalizes to
`ValidatedEmail`, `HashedPassword`, `TrustedUrl` — any case where mixing
up "raw" and "checked" values of the same underlying primitive type is
a real security bug, not just a style issue.

## Branded types for validated numeric input

```typescript
type PositiveInt = Brand<number, "PositiveInt">;

function toPositiveInt(n: number): PositiveInt {
  if (!Number.isInteger(n) || n <= 0) {
    throw new Error(`${n} is not a positive integer`);
  }
  return n as PositiveInt;
}

function paginate<T>(items: T[], pageSize: PositiveInt): T[][] {
  const pages: T[][] = [];
  for (let i = 0; i < items.length; i += pageSize) {
    pages.push(items.slice(i, i + pageSize));
  }
  return pages;
}

console.log(paginate([1, 2, 3, 4, 5], toPositiveInt(2)));

try {
  toPositiveInt(-3);
} catch (err) {
  console.log((err as Error).message);
}
```

```text
[ [ 1, 2 ], [ 3, 4 ], [ 5 ] ]
-3 is not a positive integer
```

`paginate` can't be called with a raw, unchecked `number` for
`pageSize` — a caller passing `0` or a negative number directly is
rejected by the type checker, not discovered later as an infinite loop
or a crash.

## Type-safe environment variable loading

```typescript
interface EnvConfig {
  PORT: number;
  API_KEY: string;
}

function loadEnv(env: Record<string, string | undefined>): EnvConfig {
  const port = env.PORT;
  const apiKey = env.API_KEY;
  if (!port || !apiKey) {
    throw new Error("missing required environment variables");
  }
  const parsedPort = Number(port);
  if (Number.isNaN(parsedPort)) {
    throw new Error("PORT must be a number");
  }
  return { PORT: parsedPort, API_KEY: apiKey };
}

console.log(loadEnv({ PORT: "3000", API_KEY: "secret-123" }));
```

```text
{ PORT: 3000, API_KEY: 'secret-123' }
```

`process.env` in Node is typed as `Record<string, string | undefined>`
— every value could be missing. Reading `process.env.API_KEY` directly
and passing it somewhere expecting `string` either requires a cast
(dangerous — hides a real missing-config bug) or, as here, a validating
loader that fails loudly at startup with a clear error instead of
`undefined` silently propagating into, say, an API client that then
sends requests with no auth header at all.

## Traps

**`as SafeHtml` inside `escapeHtml` is still just a cast — the brand
doesn't verify anything on its own.** If `escapeHtml`'s regex has a
bug, or if someone adds a second function that does `return raw as
SafeHtml` without actually escaping anything, the type system won't
catch it. Branding only enforces that safe values *came from* a
function you trust; it doesn't audit that function's correctness.

**Branded types don't survive JSON serialization round-trips.**
Sending a `PositiveInt` over the network as JSON and parsing it back
gives you a plain `number` again — you must re-validate at every trust
boundary (API request bodies, database reads, file reads), not just
once at the original point of creation.

**A `!port` check treats `"0"` as falsy-missing even though it's a
valid string.** `loadEnv({ PORT: "0", ... })` throws
`"missing required environment variables"` even though `"0"` is
present — a subtle trap when validating optional-but-present numeric
strings; prefer `port === undefined` over truthiness checks for this
reason.

**Secrets in `.env` files committed to version control are a runtime
problem no type system fixes.** `EnvConfig`'s types only start
protecting you *after* a value has already been loaded — they say
nothing about whether that value was safely stored or transmitted in
the first place.

## How It Actually Works

Every security-relevant example in this lesson comes back to the same structural fact: TypeScript's type checker is a **static, compile-time-only** analysis with zero runtime footprint, so it categorically cannot prevent any runtime security issue on its own — a `sanitize(input: string): string` function's type signature says nothing about whether the function's *implementation* actually strips dangerous content; the checker verifies the shape of data flowing through your program, never the semantic correctness of what a function does with it. Believing "it's typed as sanitized, so it's safe" is a type-erasure category error — the type is a label the checker attached, not a runtime-enforced property of the value.

This is also why `any` is a genuine security-relevant construct, not just a style lint: every place `any` appears (explicitly, or implicitly through `JSON.parse`, `fetch().json()`, or a missing/wrong `@types` declaration as covered in earlier lessons) is a point where the checker's structural verification is fully disabled for that value and everything derived from it — an `any`-typed value flowing into a SQL query builder or an HTML-templating call bypasses whatever type-level protections that API's own signatures might otherwise provide (like requiring a branded "sanitized string" type), because `any` is assignable to that branded type without complaint.

**Branded/nominal types** (`type SafeHtml = string & { __brand: "SafeHtml" }`) are the practical type-level defense against exactly this: intersecting a primitive with an unused, uninhabited marker property makes the checker refuse to accept a plain `string` where a `SafeHtml` is required (structurally, a plain `string` doesn't have the `__brand` property, so it's not assignable), forcing all `SafeHtml` values to originate from one sanitizing function that performs the actual `as SafeHtml` assertion — the type system enforces *that a sanitizer was called somewhere in the value's history*, which is a real, checker-verifiable guarantee, even though it still cannot verify the sanitizer's own logic is correct.

## Cheat sheet

| Pattern | Prevents |
|---|---|
| `type SafeHtml = Brand<string, "SafeHtml">` | Passing unsanitized strings where sanitized ones are required |
| `type PositiveInt = Brand<number, "PositiveInt">` | Passing unvalidated numbers into functions with numeric invariants |
| Validating `loadEnv()` instead of casting `process.env.X as string` | `undefined` config silently reaching production code |
| Re-validating branded values at trust boundaries | Assuming a brand survived serialization |
| `=== undefined` instead of falsy checks for optional strings | Rejecting valid-but-falsy values like `"0"` |

## Exercise

Add a branded `ValidatedEmail` type with a `toValidatedEmail(s: string): ValidatedEmail`
function that checks for an `@` and a `.` after it (a deliberately
simple check, not full RFC validation). Write a `sendWelcomeEmail(to: ValidatedEmail): void`
function that can only be called with validated addresses, and prove
with `tsc --noEmit` that `sendWelcomeEmail("not-an-email")` fails to
compile while `sendWelcomeEmail(toValidatedEmail("a@b.com"))` succeeds.
