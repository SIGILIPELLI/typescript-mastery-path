# 08 · Enums

An enum names a fixed set of related constant values — useful whenever a
variable should only ever hold one of a small, known list of options.

## Numeric enums (the default)

```typescript
enum Direction {
  North,   // 0
  South,   // 1
  East,    // 2
  West,    // 3
}

function move(dir: Direction): void {
  console.log(`Moving ${Direction[dir]}`);
}

move(Direction.North);   // Moving North
console.log(Direction.South);      // 1
console.log(Direction[1]);         // "South" -- numeric enums are reverse-mappable
```

By default, enum members are numbered starting at `0`. `Direction[1]` looking
up `"South"` (the reverse mapping) only works for numeric enums, not string
enums.

## String enums (usually the better default)

```typescript
enum Status {
  Pending = "PENDING",
  Active = "ACTIVE",
  Completed = "COMPLETED",
}

function describe(status: Status): string {
  switch (status) {
    case Status.Pending:
      return "Waiting to start";
    case Status.Active:
      return "In progress";
    case Status.Completed:
      return "Done";
  }
}

console.log(describe(Status.Active));   // In progress
console.log(Status.Active);              // "ACTIVE" -- readable when logged/serialized
```

String enums have no numeric reverse-mapping, but they serialize to readable
values (`"ACTIVE"` instead of `1`), which makes them much easier to debug and
safer to store in a database or send over an API.

## Custom numeric values

```typescript
enum HttpStatus {
  OK = 200,
  NotFound = 404,
  ServerError = 500,
}

function handle(status: HttpStatus): void {
  if (status === HttpStatus.OK) {
    console.log("Success");
  } else {
    console.log(`Error: ${status}`);
  }
}

handle(HttpStatus.NotFound);   // Error: 404
```

## const enums (compiled away entirely)

```typescript
const enum Size {
  Small,
  Medium,
  Large,
}

const mySize = Size.Medium;
console.log(mySize);   // 1
```

A `const enum` is inlined at compile time — the TypeScript compiler replaces
every usage with its literal value and doesn't emit any actual enum object at
runtime, producing smaller output. The tradeoff: you lose the ability to
iterate over its members or do a reverse lookup.

## Enums vs. union of string literals

```typescript
// Alternative to an enum: a union of string literal types
type StatusLiteral = "PENDING" | "ACTIVE" | "COMPLETED";

function describeLiteral(status: StatusLiteral): string {
  return status === "ACTIVE" ? "In progress" : "Other";
}

describeLiteral("ACTIVE");   // fine
// describeLiteral("DONE"); // error TS2345: not assignable to StatusLiteral
```

Many style guides (including TypeScript's own team, in recent guidance) now
prefer string literal unions over enums for simple cases — they add zero
runtime code, and TypeScript's structural typing already gives you the same
compile-time safety. Reach for a real `enum` when you specifically want a
namespaced group of related constants (`Status.Active` reads more clearly
than a bare `"ACTIVE"` string floating around your code).

## How It Actually Works

Unlike almost everything else covered so far, a numeric or string `enum` is **not** fully erased — it's one of the few TypeScript constructs that generates real runtime JavaScript. `enum Direction { Up, Down }` compiles to an IIFE that builds an object with both forward mappings (`Direction.Up === 0`) and, for numeric enums only, *reverse* mappings (`Direction[0] === "Up"`), so the runtime object literally has twice as many keys as members you declared. String enums skip the reverse mapping (a string value doesn't need a numeric-index lookup back to its name), so they produce a smaller, plainer object.

This runtime cost is exactly why the language later added `as const` objects as a lighter alternative: `const Direction = { Up: "UP", Down: "DOWN" } as const` produces one plain object with no IIFE wrapper and no reverse-mapping bloat, and pairing it with `type Direction = typeof Direction[keyof typeof Direction]` gets you a structurally-equivalent union-of-literal-values type for compile-time checking, with strictly less emitted code. The checker treats a `keyof typeof` union exactly like any other string-literal union for narrowing purposes — `switch` exhaustiveness checks work identically on both.

`const enum` is a separate, more aggressive variant: annotate `enum` with `const` and the compiler *inlines* every reference at each use site (`Direction.Up` becomes the literal `0` in the emitted code) and, given `isolatedModules` is off, emits no runtime object for the enum at all — the fastest option, but it breaks tools that compile files independently (like Babel or `ts-node` in isolated-transpile modes) because inlining requires cross-file type information the const-enum's own module doesn't carry, which is why `isolatedModules: true` (required by most modern bundlers) forbids `const enum` outright.

## Cheat sheet

| Kind | When to use |
|------|-------------|
| Numeric enum | Rarely needed now; reverse-mapping is the main reason to pick it |
| String enum | Readable, serializable — a solid default when you want an enum |
| `const enum` | Same as string/numeric, but compiled away for smaller output |
| String literal union | Zero runtime cost, often simpler for small fixed sets |

## Exercise

Define a string enum `Direction` with `Up`, `Down`, `Left`, `Right` (values
`"UP"`, `"DOWN"`, `"LEFT"`, `"RIGHT"`). Write a function `opposite(dir:
Direction): Direction` that returns the opposite direction using a `switch`.
Then rewrite the same logic using a string literal union type instead of the
enum, and compare the two approaches.
