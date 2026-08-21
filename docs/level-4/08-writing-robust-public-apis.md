# 08 · Writing Robust Public APIs

Code you call from the same file behaves however you want; code
consumed by *other people's* codebases needs a stable, predictable
surface — clear error handling that doesn't force `try`/`catch`
everywhere, overloaded signatures for genuinely different call shapes,
and return types that can't be silently mutated by a caller. This
module covers three patterns common in well-designed TypeScript
libraries.

## The `Result` type: errors as values, not exceptions

```typescript
type Result<T, E = Error> =
  | { ok: true; value: T }
  | { ok: false; error: E };

function ok<T>(value: T): Result<T, never> {
  return { ok: true, value };
}
function err<E>(error: E): Result<never, E> {
  return { ok: false, error };
}

interface ParseError {
  message: string;
  position: number;
}

function parseCount(input: string): Result<number, ParseError> {
  const trimmed = input.trim();
  if (trimmed === "") {
    return err({ message: "empty input", position: 0 });
  }
  const n = Number(trimmed);
  if (Number.isNaN(n)) {
    return err({ message: `"${trimmed}" is not a number`, position: 0 });
  }
  if (n < 0) {
    return err({ message: "count cannot be negative", position: 0 });
  }
  return ok(n);
}

function handle(input: string): void {
  const result = parseCount(input);
  if (result.ok) {
    console.log("parsed:", result.value);
  } else {
    console.log("failed:", result.error.message);
  }
}

handle("42");
handle("-5");
handle("abc");
```

```text
parsed: 42
failed: count cannot be negative
failed: "abc" is not a number
```

`parseCount`'s **signature itself** documents every way it can fail —
a caller reading the type doesn't need to hunt through the
implementation or documentation to discover what exceptions might be
thrown. `if (result.ok)` narrows `result` to the success branch;
`else` narrows it to the error branch — TypeScript enforces you check
which one you have before accessing `.value` or `.error`, unlike a
thrown exception a caller can simply forget to catch.

## Overloaded signatures for meaningfully different call shapes

```typescript
function createLogger(name: string): { log(msg: string): void };
function createLogger(name: string, level: "debug" | "info" | "error"): { log(msg: string): void };
function createLogger(
  name: string,
  level: "debug" | "info" | "error" = "info"
): { log(msg: string): void } {
  return {
    log(msg: string) {
      console.log(`[${level}] ${name}: ${msg}`);
    },
  };
}

const logger1 = createLogger("api");
logger1.log("started");
const logger2 = createLogger("db", "debug");
logger2.log("query executed");
```

```text
[info] api: started
[debug] db: query executed
```

The two signatures above the implementation are what callers and
autocomplete actually see; the implementation signature (with the
default parameter) is invisible from outside. Overloads are worth
reaching for specifically when the *meaning* of a call changes based on
arity — a single signature with `level?: "debug" | "info" | "error"`
would work identically here, so overloads are somewhat redundant in
this exact example, but become essential once, say, a two-argument
call and a three-argument call return genuinely *different types*
(which a single optional-parameter signature can't express).

## Readonly return types prevent silent mutation

```typescript
interface PublicApiOptions {
  readonly timeout: number;
  readonly retries: number;
}

function getDefaultOptions(): Readonly<PublicApiOptions> {
  return { timeout: 5000, retries: 3 };
}

const opts = getDefaultOptions();
console.log(opts);
// opts.timeout = 1000; // compile error if uncommented
```

```text
{ timeout: 5000, retries: 3 }
```

Without `readonly`, a caller doing
`const opts = getDefaultOptions(); opts.timeout = 1000;` compiles fine
and silently mutates what might be a shared/cached default-options
object — a classic "why did changing this in one place affect an
unrelated part of the app" bug. `readonly` at the type level catches
the mutation attempt at the call site, before it ever runs.

## Traps

**Return-type inference on `ok()`/`err()` widens if you drop the
explicit annotations.** Removing `: Result<T, never>` from `ok`'s
signature lets TypeScript infer a looser type from the function body,
and downstream discriminated-union narrowing (`if (result.ok) {...}
else {...}`) can stop working correctly — always annotate the return
type explicitly on small helper constructors like this in a public API,
even though TypeScript could technically infer something.

**A `Result<T, E>` API doesn't stop a caller from ignoring the
`.error` case entirely** if they just do `result.value` without
narrowing first — that's a compile error only when `strict` (specifically
`strictNullChecks`) is on. A public library targeting consumers who
might not use `strict` mode should document this clearly, since the
whole safety guarantee depends on the caller's own compiler settings.

**Overloads are checked in declaration order, and the *last* matching
overload's parameter types are what the implementation body sees** —
if two overload signatures could both match a given call, TypeScript
picks the first one that fits, which can produce a confusingly
"wrong" overload being selected if they're ordered loosely-to-strict
instead of strictly-to-loose.

**`readonly` is shallow.** `Readonly<PublicApiOptions>` prevents
reassigning `opts.timeout`, but if `PublicApiOptions` had a nested
object property, that nested object's own fields would still be
mutable unless you also apply `Readonly` (or a recursive `DeepReadonly`
utility) to it.

## Cheat sheet

| Pattern | Use for |
|---|---|
| `Result<T, E>` | Public functions where callers must handle failure explicitly |
| Function overloads | Different argument counts/types that mean genuinely different things |
| `Readonly<T>` return type | Preventing accidental mutation of shared/default objects |
| Explicit return-type annotations on helpers | Keeping discriminated unions narrow and correct |
| `DeepReadonly<T>` (custom) | Full immutability for nested return values |

## Exercise

Add an `unwrapOr(result: Result<T, E>, fallback: T): T` helper function
(generic over both `T` and `E`) that returns `result.value` if `ok`,
otherwise the fallback. Use it to rewrite `handle` so a failed parse
falls back to `0` instead of just logging the error, and verify with
`tsc --noEmit --strict` that `unwrapOr`'s generic parameters are
correctly inferred at each call site without any explicit type
arguments.
