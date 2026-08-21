# 01 · Advanced Type-Level Programming

TypeScript's type system is Turing-complete-ish in practice: recursive
conditional types, template literal types, and tuple manipulation let
you compute types, not just describe them. This module builds a
dot-path type (`"server.port"`) that autocompletes and type-checks
against a real object shape, plus the smaller building blocks it rests
on.

## Template literal types

```typescript
type EventName<T extends string> = `on${Capitalize<T>}`;
type ClickEvent = EventName<"click">; // "onClick"
```

Template literal types build new string-literal types out of others —
`Capitalize` is one of TypeScript's four built-in intrinsic string
manipulation types (`Uppercase`, `Lowercase`, `Capitalize`,
`Uncapitalize`), all resolved entirely at compile time.

## Recursive conditional types: splitting a string

```typescript
type Split<S extends string, D extends string> =
  S extends `${infer Head}${D}${infer Rest}`
    ? [Head, ...Split<Rest, D>]
    : [S];

type Parts = Split<"a.b.c", ".">; // ["a", "b", "c"]
```

`infer` inside a conditional type's `extends` clause captures a piece of
the matched string as its own type variable. `Split` recurses on `Rest`
until the pattern no longer matches, at which point the base case `[S]`
wraps whatever's left — the same head-recursion shape as splitting a
string at runtime, just operating on types instead of values.

## Building a type-safe dot-path

```typescript
type Paths<T> = T extends object
  ? {
      [K in keyof T & string]: T[K] extends object
        ? K | `${K}.${Paths<T[K]>}`
        : K;
    }[keyof T & string]
  : never;

interface Config {
  server: { port: number; host: string };
  db: { url: string };
}
type ConfigPath = Paths<Config>;
// "server" | "server.port" | "server.host" | "db" | "db.url"

function getPath<T, P extends Paths<T>>(obj: T, path: P): unknown {
  const parts = (path as string).split(".");
  let current: unknown = obj;
  for (const part of parts) {
    current = (current as Record<string, unknown>)[part];
  }
  return current;
}

const config: Config = {
  server: { port: 8080, host: "localhost" },
  db: { url: "postgres://" },
};
console.log(getPath(config, "server.port"));
console.log(getPath(config, "db.url"));
```

```text
8080
postgres://
```

`getPath(config, "server.pot")` (a typo) is a **compile error**, not a
runtime `undefined` — `Paths<Config>` is a closed union of every valid
dot-path through `Config`, computed once from the interface, so a typo
falls outside the union and TypeScript rejects it before the code ever
runs. This is the mechanism behind libraries like Zod's
`.path()` helpers and typed form libraries' field-path props.

## Tuple-based type-level arithmetic

```typescript
type BuildTuple<L extends number, T extends unknown[] = []> = T["length"] extends L
  ? T
  : BuildTuple<L, [...T, unknown]>;
type Add<A extends number, B extends number> = [...BuildTuple<A>, ...BuildTuple<B>]["length"];

type Sum = Add<3, 4>; // 7
const sum: Sum = 7;
console.log(sum);
```

```text
7
```

There's no arithmetic operator on numeric types in TypeScript — `Add`
"computes" 3+4 by building a 3-element tuple and a 4-element tuple out
of `unknown`, concatenating them, and reading off the resulting tuple's
`length`. This works but doesn't scale: it's O(n) in tuple length, and
large numbers blow the recursion depth limit (see the trap below).

## Exhaustiveness checking with `never`

```typescript
function assertNever(x: never): never {
  throw new Error(`Unexpected value: ${JSON.stringify(x)}`);
}

type Shape = { kind: "circle"; r: number } | { kind: "square"; s: number };
function area(shape: Shape): number {
  switch (shape.kind) {
    case "circle":
      return Math.PI * shape.r ** 2;
    case "square":
      return shape.s ** 2;
    default:
      return assertNever(shape);
  }
}
console.log(area({ kind: "circle", r: 2 }).toFixed(2));
```

```text
12.57
```

Inside `default`, `shape`'s type has been narrowed to `never` — every
member of the `Shape` union was handled by an earlier `case`. If someone
adds a third shape kind later and forgets to add its `case`, `shape` in
the `default` branch is no longer `never` (it's the new, unhandled
variant), and passing it to `assertNever(x: never)` becomes a compile
error at exactly the spot the new case was missed.

## Traps

**Deep recursion hits `TS2589`.** `Add<500, 500>` (or a `Paths<T>` over
a very deeply nested object) can exceed TypeScript's recursion depth
limit: `Type instantiation is excessively deep and possibly infinite`.
Tuple-based arithmetic in particular is unusable much past small numbers
— it exists here to teach the mechanism, not as production advice for
real math.

**`Paths<T>` recurses into arrays and functions too**, since they're
also `object` — without excluding them (`T[K] extends object ? ... `
needs a check like `T[K] extends readonly unknown[] ? K : ...`), a
`string[]` property produces nonsensical numeric-index paths. Real
implementations (as in `lodash`'s type-level `get` helpers) special-case
arrays and functions explicitly.

**`infer` can only appear inside the `extends` clause of a conditional
type**, not in an arbitrary position — `type X<S> = S extends infer T ? T : never`
is valid, but trying to use `infer` outside a conditional's `extends`
position is a syntax error.

## Cheat sheet

| Feature | What it does |
|---|---|
| `` `${A}${B}` `` (template literal type) | Concatenates string-literal types |
| `Capitalize<T>` / `Uppercase` / `Lowercase` / `Uncapitalize` | Built-in string-literal transforms |
| `S extends \`${infer Head}${D}${infer Rest}\`` | Captures parts of a matched string type |
| `T extends unknown[] ? T["length"] : never` | Reads a tuple's length as a numeric literal type |
| `function assertNever(x: never): never` | Compile-time exhaustiveness check |

## Exercise

Extend `Paths<T>` with a companion `PathValue<T, P extends Paths<T>>`
type that resolves to the *value type* at that path (e.g.
`PathValue<Config, "server.port">` should be `number`, not `unknown`).
Rewrite `getPath` to use it as its return type instead of `unknown`, and
confirm `getPath(config, "server.port")` now has inferred type `number`
by assigning its result to a `const port: number = ...` without a cast.
