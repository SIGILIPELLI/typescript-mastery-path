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
