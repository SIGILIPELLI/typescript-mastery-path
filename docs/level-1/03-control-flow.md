# 03 · Control Flow with Types

Control flow in TypeScript looks just like JavaScript, but the compiler
tracks types through every branch — narrowing, widening, and flagging
unreachable code along the way.

## `if` / `else` and type narrowing

```typescript
function formatId(id: string | number): string {
  if (typeof id === "number") {
    return id.toFixed(0);       // narrowed to number in this branch
  } else {
    return id.toUpperCase();    // narrowed to string in this branch
  }
}

console.log(formatId(42));     // "42"
console.log(formatId("ab12")); // "AB12"
```

Inside the `if (typeof id === "number")` block, TypeScript *knows* `id` is a
`number` — it narrows the union `string | number` down to just `number` for
that block, and back to `string` in the `else`. This is TypeScript's **control
flow analysis**, and it's why type-safe code often needs fewer manual casts
than you'd expect.

## `switch` statements

```typescript
type Direction = "north" | "south" | "east" | "west";

function turnRight(direction: Direction): Direction {
  switch (direction) {
    case "north":
      return "east";
    case "east":
      return "south";
    case "south":
      return "west";
    case "west":
      return "north";
  }
}

console.log(turnRight("north"));   // "east"
```

`Direction` here is a **string literal union** (covered fully in
[Module 5](05-interfaces-type-aliases.md)) — a type restricted to exactly
those four strings. Because every possible value is handled, TypeScript can
verify the `switch` is exhaustive; if a case is missing, tools like
`noImplicitReturns` (see [Module 9](09-modules-tsconfig.md)) will flag a
missing `return`.

## Truthiness narrowing

```typescript
function printLength(text: string | null) {
  if (text) {
    console.log(text.length);   // narrowed: text is string here (not null, not "")
  } else {
    console.log("nothing to print");
  }
}

printLength("hello");   // 5
printLength(null);      // nothing to print
printLength("");        // nothing to print -- empty string is falsy too!
```

Truthiness narrowing is convenient but coarse — it also excludes `0`, `""`,
and `NaN`, which are often valid values. When zero or empty-string need to be
treated as "present", check explicitly instead:

```typescript
function printCount(count: number | null) {
  if (count !== null) {
    console.log(`Count: ${count}`);   // handles count === 0 correctly
  }
}
```

## Loops

```typescript
const scores: number[] = [72, 88, 95, 60];

// for...of -- iterate values, most common for arrays
for (const score of scores) {
  console.log(score);
}

// for...in -- iterate keys (mostly for objects, rarely arrays)
const inventory: Record<string, number> = { apples: 5, bananas: 3 };
for (const item in inventory) {
  console.log(`${item}: ${inventory[item]}`);
}

// classic for -- when you need the index
for (let i = 0; i < scores.length; i++) {
  console.log(`#${i}: ${scores[i]}`);
}

// while / do...while
let remaining = 3;
while (remaining > 0) {
  console.log(`${remaining} left`);
  remaining--;
}
```

## `break` and `continue`

```typescript
const numbers = [1, 2, 3, 4, 5, 6];

for (const n of numbers) {
  if (n % 2 !== 0) {
    continue;   // skip odd numbers
  }
  if (n > 4) {
    break;      // stop once we pass 4
  }
  console.log(n);   // 2, 4
}
```

## Ternary expressions and typed results

```typescript
function classify(score: number): string {
  return score >= 60 ? "pass" : "fail";
}

// TypeScript infers a union return type when branches differ:
function describe(x: number) {
  return x > 0 ? "positive" : x < 0 ? "negative" : "zero";
  // inferred return type: string (all branches are strings here)
}
```

## `never` — the type of "this can't happen"

`never` represents a value that can never occur — a function that always
throws, or an exhaustive `switch` with no remaining cases:

```typescript
function assertUnreachable(x: never): never {
  throw new Error(`Unexpected value: ${x}`);
}

function area(shape: { kind: "circle"; radius: number } | { kind: "square"; side: number }): number {
  switch (shape.kind) {
    case "circle":
      return Math.PI * shape.radius ** 2;
    case "square":
      return shape.side ** 2;
    default:
      // If a new shape variant is added later without updating this switch,
      // TypeScript will error here because `shape` won't be assignable to `never`.
      return assertUnreachable(shape);
  }
}
```

This pattern — an exhaustiveness check via `never` — is one of TypeScript's
most valuable safety nets for `switch` statements over a fixed set of cases.

## Cheat sheet

| Construct | Notes |
|-----------|-------|
| `if`/`else` | Narrows union types per-branch via control flow analysis |
| `switch` | Same narrowing; combine with `never` for exhaustiveness checks |
| Truthiness check | Excludes `0`, `""`, `NaN`, `null`, `undefined` -- be explicit if 0/"" are valid |
| `for...of` | Iterate values (arrays, strings, most iterables) |
| `for...in` | Iterate keys (objects) |
| `never` | Type of a value that can't occur -- powers exhaustiveness checks |

## Exercise

Write a type `TrafficLight = "red" | "yellow" | "green"` and a function
`next(light: TrafficLight): TrafficLight` that returns the next light in the
cycle (red → green → yellow → red) using a `switch`, with an `assertUnreachable`
default case for exhaustiveness. Then write a function
`sumPositive(numbers: number[]): number` that uses a `for...of` loop with
`continue` to skip non-positive numbers and returns the sum of the rest.
