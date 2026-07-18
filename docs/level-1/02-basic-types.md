# 02 · Basic Types

TypeScript's type system layers compile-time checks on top of JavaScript's
existing runtime types. Most basic types read almost like English.

## `string`, `number`, `boolean`

```typescript
let username: string = "alice";
let age: number = 30;
let isActive: boolean = true;

// TypeScript infers types from the initializer even without an annotation
let city = "Seattle";   // inferred as string
// city = 42;           // error TS2322: Type 'number' is not assignable to type 'string'
```

Unlike some languages, TypeScript has a single `number` type for all numbers —
no separate `int`/`float`/`double`. There's also `bigint` for arbitrarily large
integers (`const big: bigint = 123n;`).

## Arrays

Two equivalent syntaxes for typing arrays:

```typescript
let scores: number[] = [90, 85, 77];
let names: Array<string> = ["Alice", "Bob"];   // generic syntax, same meaning

scores.push(100);     // fine -- number
// scores.push("A");  // error TS2345: Argument of type 'string' is not assignable
//                        to parameter of type 'number'.
```

`T[]` and `Array<T>` are identical — `T[]` is more common for simple element
types, `Array<T>` reads more clearly when `T` itself is complex.

## Tuples — fixed-length, mixed-type arrays

A tuple is an array with a known length and a known type at each position —
useful for representing a fixed, small, ordered set of values:

```typescript
let point: [number, number] = [10, 20];
let entry: [string, number, boolean] = ["Alice", 30, true];

console.log(point[0]);   // 10 -- typed as number
console.log(entry[0]);   // "Alice" -- typed as string
console.log(entry[1]);   // 30 -- typed as number

// entry[0] = 42;   // error -- position 0 must be string
// point.push(30);  // TypeScript allows this at compile time (a known tuple
//                     limitation) but it's discouraged -- avoid mutating tuples
```

Tuples with named positions (labels are documentation only, purely for
readability — they don't change behavior):

```typescript
let httpResponse: [status: number, body: string] = [200, "OK"];
```

## `any` — opting out of type checking

`any` disables type checking entirely for that value — TypeScript will let you
do anything with it, and won't catch mistakes:

```typescript
let data: any = 42;
data = "now a string";      // fine, any allows reassignment to any type
data.toUpperCase();          // no error at compile time...
data();                      // ...even this "compiles" -- calling a number as a function!
// Runtime: TypeError: data is not a function
```

`any` is an escape hatch for gradually migrating JavaScript code or handling
truly dynamic data — but overusing it defeats the entire purpose of
TypeScript. Treat every `any` in a codebase as a small hole in your safety net.

## `unknown` — the type-safe alternative to `any`

`unknown` also accepts any value, but forces you to **narrow** the type before
doing anything with it — the safe way to handle values whose type you don't
know yet (e.g. parsed JSON, user input):

```typescript
let value: unknown = 42;

// value.toUpperCase();   // error TS18046: 'value' is of type 'unknown'.

if (typeof value === "number") {
  console.log(value.toFixed(2));   // fine -- narrowed to number inside this block
}

if (typeof value === "string") {
  console.log(value.toUpperCase());   // fine -- narrowed to string here instead
}
```

Type narrowing gets its own deep dive in [Level 2, Module 9](../level-2/09-type-narrowing-guards.md).
For now: **prefer `unknown` over `any`** whenever a value's type is genuinely
not known up front — it keeps the compiler checking your work instead of
trusting you blindly.

## `null` and `undefined`

```typescript
let a: null = null;
let b: undefined = undefined;

let maybeName: string | null = null;   // union type -- either a string or null
maybeName = "Alice";                    // also fine
```

With `strict` mode on (the default recommendation, see [Module 9](09-modules-tsconfig.md)),
`null` and `undefined` are **not** automatically assignable to other types —
you must explicitly include them in a union, as shown above. This single
setting (`strictNullChecks`) eliminates an entire class of "cannot read
property of undefined" runtime crashes by forcing you to handle the
possibility at compile time.

## `void` — functions that return nothing

```typescript
function logMessage(message: string): void {
  console.log(message);
  // no return statement -- or a bare `return;` -- both fine for void
}
```

## Type inference vs. explicit annotation

TypeScript infers types wherever it reasonably can — you don't need to
annotate every single variable:

```typescript
let count = 0;              // inferred: number
let label = "score";        // inferred: string
let items = [1, 2, 3];       // inferred: number[]

// Explicit annotations matter most for function parameters (TypeScript can't
// guess what a caller will pass) and for declaring intent on public APIs.
```

## Cheat sheet

| Type | Example | Notes |
|------|---------|-------|
| `string` | `"hello"` | Text |
| `number` | `42`, `3.14` | All numbers -- no int/float split |
| `boolean` | `true`, `false` | |
| `T[]` / `Array<T>` | `number[]` | Homogeneous array |
| `[T, U]` | `[string, number]` | Fixed-length tuple |
| `any` | -- | Disables checking -- avoid |
| `unknown` | -- | Safe "don't know yet" -- requires narrowing |
| `null` / `undefined` | -- | Only assignable elsewhere via a union under `strict` |
| `void` | -- | Function returns nothing meaningful |

## Exercise

Declare typed variables for a `productName` (`string`), `price` (`number`),
`inStock` (`boolean`), and a `tags` array of strings. Then create a tuple
`priceRange: [number, number]` for a min/max price. Write a function
`describeValue(value: unknown): string` that uses `typeof` checks to return a
different description string depending on whether `value` is a `string`, a
`number`, a `boolean`, or none of those.
