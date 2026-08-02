# 09 · Type Narrowing & Type Guards

Several earlier modules leaned on narrowing without naming it directly —
the `typeof` check in [Module 1](01-union-intersection-types.md), the
`instanceof Error` check in [Module 6](06-async-await-types.md), the
hand-written `isTodo` guard in [Module 8](08-working-with-json-apis.md).
This module makes the pattern explicit: how TypeScript narrows a broad
type down to a specific one within a block of code, and how to write your
own narrowing functions when the built-in checks aren't enough.

## What narrowing actually is

Narrowing is the compiler tracking, statement by statement, that a
variable's type has become more specific than its declared type — purely
from the control flow you wrote, with no extra annotations needed:

```typescript
function describeValue(value: string | number | boolean): string {
  if (typeof value === "string") {
    return `string of length ${value.length}`;   // value: string here
  }
  if (typeof value === "number") {
    return `number: ${value.toFixed(2)}`;         // value: number here
  }
  return `boolean: ${value ? "true" : "false"}`;  // value: boolean here
  // (the only type left after excluding string and number)
}

console.log(describeValue("hi"));      // string of length 2
console.log(describeValue(3.14159));   // number: 3.14
console.log(describeValue(true));      // boolean: true
```

!!! note "Naming clash to watch for"
    `describe` is a fine name in isolation, but if your project has Jest's
    global types in scope (`@types/jest`), it collides with Jest's global
    `describe` function used for grouping tests — the compiler will
    reject a top-level `function describe(...)` as a duplicate
    identifier. This is the same category of trap as the `Comment` /
    DOM-global collision from [Module 1](01-union-intersection-types.md):
    watch for names that overlap with globals your type setup already
    provides.

## `typeof` narrowing

Works for JavaScript's primitive types: `"string"`, `"number"`,
`"boolean"`, `"undefined"`, `"function"`, `"object"`, `"symbol"`,
`"bigint"`.

```typescript
function formatValue(value: string | string[] | undefined): string {
  if (typeof value === "undefined") {
    return "(none)";
  }
  if (typeof value === "string") {
    return value;
  }
  return value.join(", ");   // last remaining option: string[]
}

console.log(formatValue(undefined));           // (none)
console.log(formatValue("solo"));              // solo
console.log(formatValue(["a", "b", "c"]));     // a, b, c
```

Note `typeof value === "object"` would be a trap for narrowing out
`string[]` specifically, since arrays and `null` both report `"object"`
from `typeof` — it isn't precise enough to distinguish an array from a
plain object or `null`.

## `instanceof` narrowing

For distinguishing classes:

```typescript
class NotFoundError extends Error {}
class ValidationError extends Error {
  constructor(message: string, public field: string) {
    super(message);
  }
}

function handle(error: Error): string {
  if (error instanceof ValidationError) {
    return `Invalid field "${error.field}": ${error.message}`;
  }
  if (error instanceof NotFoundError) {
    return `Not found: ${error.message}`;
  }
  return `Unexpected error: ${error.message}`;
}

console.log(handle(new ValidationError("Required", "email")));
// Invalid field "email": Required
console.log(handle(new NotFoundError("User 42")));
// Not found: User 42
```

Check the more specific subclass before the more general one when classes
share a hierarchy — `instanceof Error` would match both `ValidationError`
and `NotFoundError`, so it has to come last (or not at all, here).

## `in` narrowing

Checks for the presence of a property, useful for unions of object shapes
that don't have an explicit discriminant field:

```typescript
interface Car {
  wheels: 4;
  drive(): string;
}

interface Boat {
  propellers: number;
  sail(): string;
}

function operate(vehicle: Car | Boat): string {
  if ("drive" in vehicle) {
    return vehicle.drive();   // narrowed to Car
  }
  return vehicle.sail();      // narrowed to Boat
}

const myCar: Car = { wheels: 4, drive: () => "Vroom" };
const myBoat: Boat = { propellers: 2, sail: () => "Whoosh" };

console.log(operate(myCar));    // Vroom
console.log(operate(myBoat));   // Whoosh
```

## Discriminated union narrowing

Covered fully in [Module 1](01-union-intersection-types.md) — a quick
reminder, since it's the most common narrowing pattern in real code:

```typescript
type Loading = { status: "loading" };
type Success = { status: "success"; data: string };
type Failure = { status: "failure"; error: string };
type RequestState = Loading | Success | Failure;

function render(state: RequestState): string {
  switch (state.status) {
    case "loading":
      return "Loading...";
    case "success":
      return `Data: ${state.data}`;
    case "failure":
      return `Error: ${state.error}`;
  }
}

console.log(render({ status: "loading" }));                       // Loading...
console.log(render({ status: "success", data: "42 users" }));     // Data: 42 users
console.log(render({ status: "failure", error: "Timeout" }));     // Error: Timeout
```

## User-defined type guards

When built-in checks aren't precise enough, write a function that returns
a **type predicate** (`value is SomeType`) instead of a plain `boolean`:

```typescript
interface Cat {
  kind: "cat";
  meow(): string;
}

interface Dog {
  kind: "dog";
  bark(): string;
}

function isCat(animal: Cat | Dog): animal is Cat {
  return animal.kind === "cat";
}

function makeSound(animal: Cat | Dog): string {
  if (isCat(animal)) {
    return animal.meow();   // narrowed to Cat by the type predicate
  }
  return animal.bark();     // narrowed to Dog
}

const tom: Cat = { kind: "cat", meow: () => "Meow!" };
console.log(makeSound(tom));   // Meow!
```

The return type `animal is Cat` is what makes this a *type guard*
function rather than an ordinary boolean-returning function — calling it
inside an `if` teaches the compiler the narrowing, exactly like a
built-in `typeof` or `instanceof` check would.

### Guards for validating truly unknown data

The same pattern scales down to `unknown`, which is the realistic
starting point for anything parsed from JSON or user input:

```typescript
interface Coordinates {
  lat: number;
  lng: number;
}

function isCoordinates(value: unknown): value is Coordinates {
  if (typeof value !== "object" || value === null) {
    return false;
  }
  const candidate = value as Record<string, unknown>;
  return (
    typeof candidate.lat === "number" &&
    typeof candidate.lng === "number"
  );
}

function describeLocation(value: unknown): string {
  if (!isCoordinates(value)) {
    return "Invalid coordinates";
  }
  return `(${value.lat}, ${value.lng})`;   // narrowed to Coordinates
}

console.log(describeLocation({ lat: 12.9, lng: 77.6 }));   // (12.9, 77.6)
console.log(describeLocation({ lat: "north" }));           // Invalid coordinates
console.log(describeLocation(null));                       // Invalid coordinates
```

## A trap: a type guard that lies still compiles

TypeScript trusts a type predicate's return type completely — it never
verifies that the function's *body* actually matches what it claims:

```typescript
function isCatButWrong(animal: Cat | Dog): animal is Cat {
  return true;   // always returns true, regardless of the real shape
}

function unsafeMakeSound(animal: Cat | Dog): string {
  if (isCatButWrong(animal)) {
    return animal.meow();   // TypeScript believes this is safe -- it compiles
  }
  return animal.bark();
}

const rex: Dog = { kind: "dog", bark: () => "Woof!" };
// unsafeMakeSound(rex);
// Runtime error: animal.meow is not a function
// TypeScript gave zero warning, because it never checks a type guard's
// actual logic against its claimed predicate -- only that the function
// returns a `boolean`-compatible value.
```

A wrong type guard is worse than no type guard at all, because it looks
just as trustworthy to the type checker as a correct one. Always test
guard functions directly (see [Module 7](07-testing-jest.md)) against
both matching and non-matching inputs.

## Narrowing pitfall: reassignment can widen back inside closures

If a variable is only ever read after being narrowed, TypeScript is smart
enough to carry the narrowed type into a nested closure. But the moment
that variable is reassigned *anywhere* in the enclosing scope, TypeScript
can no longer trust the narrowing inside a callback that might run later
— it widens back to the full original type there, even before the point
where the reassignment happens:

```typescript
function processValue(input: string | number): void {
  let value = input;

  if (typeof value === "string") {
    setTimeout(() => {
      value.toUpperCase();
      // error TS2339: Property 'toUpperCase' does not exist on type
      // 'string | number'. Property 'toUpperCase' does not exist on
      // type 'number'.
      // Even though `value` is provably a string at the moment
      // `setTimeout` is called, the later `value = 42` means TypeScript
      // can no longer guarantee what `value` holds by the time this
      // callback actually runs -- so inside the closure, it's back to
      // the full `string | number` type.
    }, 0);

    value = 42;   // <- this reassignment is what breaks the narrowing above
  }
}
```

The fix is to re-check inside the closure (`if (typeof value ===
"string") { ... }` again) or to capture the narrowed value in its own
`const` before the closure is created, so nothing later in the function
can invalidate it:

```typescript
function processValueFixed(input: string | number): void {
  if (typeof input === "string") {
    const narrowed = input;   // a fresh `const`, never reassigned anywhere
    setTimeout(() => {
      console.log(narrowed.toUpperCase());   // safe -- `narrowed` can't change
    }, 0);
  }
}

processValueFixed("hello");   // logs "HELLO" after the timeout fires
```

Narrowing is a static-analysis convenience tied to the compiler's ability
to prove a variable's value can't change before it's used — not a
persistent runtime fact. A plain, never-reassigned binding keeps its
narrowed type anywhere, including inside closures; a `let` that gets
reassigned later loses that guarantee inside any closure created before
the reassignment.

## Cheat sheet

| Guard | Example | Best for |
|---|---|---|
| `typeof` | `typeof x === "string"` | Primitives |
| `instanceof` | `x instanceof MyClass` | Class instances, check subclasses before base classes |
| `in` | `"prop" in x` | Distinguishing object shapes without a discriminant |
| Discriminated union | `switch (x.kind) { ... }` | Unions with a shared literal tag field |
| `Array.isArray` | `Array.isArray(x)` | Distinguishing an array from other objects |
| User-defined guard | `function isX(v): v is X { ... }` | Anything the built-ins can't express directly |
| `value is T` return type | required for a custom guard | Without it, TS treats the function as a plain `boolean` and narrows nothing |

## Exercise

Write a discriminated union `ApiEvent = { type: "connected"; sessionId:
string } | { type: "message"; text: string } | { type: "disconnected";
reason: string }`. Write a function `logEvent(event: ApiEvent): void`
that narrows on `type` and logs an appropriately formatted line for each
case. Then write a standalone user-defined type guard `isApiEvent(value:
unknown): value is ApiEvent` that checks for a valid `type` field before
trusting the rest of the shape, and use it to filter a mixed array of
`unknown` values down to only the valid `ApiEvent` objects.
