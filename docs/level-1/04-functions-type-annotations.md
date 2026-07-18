# 04 · Functions & Type Annotations

Functions are where TypeScript's type checking earns its keep the most —
every parameter and return value gets verified at every call site.

## Parameter and return types

```typescript
function add(a: number, b: number): number {
  return a + b;
}

add(2, 3);        // 5
// add(2, "3");    // error TS2345: Argument of type 'string' is not
//                    assignable to parameter of type 'number'.
```

TypeScript usually infers the return type from the function body, but
annotating it explicitly (as above) is good practice for any function other
code depends on — it documents intent and catches mistakes if the
implementation later drifts from what callers expect.

## Arrow functions

```typescript
const multiply = (a: number, b: number): number => a * b;

const greet = (name: string): string => {
  return `Hello, ${name}`;
};

console.log(multiply(4, 5));   // 20
console.log(greet("Sam"));     // Hello, Sam
```

## Optional parameters

An optional parameter (`?`) may be omitted by the caller — its type
automatically includes `undefined`:

```typescript
function buildGreeting(name: string, title?: string): string {
  if (title) {
    return `Hello, ${title} ${name}`;
  }
  return `Hello, ${name}`;
}

console.log(buildGreeting("Smith"));           // Hello, Smith
console.log(buildGreeting("Smith", "Dr."));    // Hello, Dr. Smith
```

Optional parameters must come after all required parameters — TypeScript
enforces this at the declaration site.

## Default parameters

A default value makes a parameter optional *and* gives it a fallback,
avoiding manual `if (x === undefined)` checks:

```typescript
function power(base: number, exponent: number = 2): number {
  return base ** exponent;
}

console.log(power(5));       // 25 -- exponent defaults to 2
console.log(power(5, 3));    // 125
```

## Rest parameters

```typescript
function sum(...numbers: number[]): number {
  return numbers.reduce((total, n) => total + n, 0);
}

console.log(sum(1, 2, 3));         // 6
console.log(sum(10, 20, 30, 40));  // 100
```

`...numbers: number[]` collects any number of trailing arguments into a
typed array — the type annotation applies to each element, not the rest
parameter as a whole.

## Function types

A function's *type* — its parameter and return types, independent of any
particular implementation — can be written and reused like any other type:

```typescript
// A variable whose value must be a function taking two numbers, returning a number
let operation: (a: number, b: number) => number;

operation = (a, b) => a + b;    // fine -- parameter types inferred from the annotation
console.log(operation(3, 4));   // 7

operation = (a, b) => a - b;    // reassigning is fine, shape still matches
console.log(operation(3, 4));   // -1
```

This is especially useful for parameters that accept a callback:

```typescript
function applyOperation(a: number, b: number, op: (x: number, y: number) => number): number {
  return op(a, b);
}

console.log(applyOperation(6, 3, (x, y) => x / y));   // 2
```

## Function overloads

When a function's behavior genuinely differs by argument shape (not just a
union of types in one signature), you can declare multiple **call
signatures** followed by a single implementation that handles them all:

```typescript
// Overload signatures -- describe the valid ways to call this function
function makeDate(timestamp: number): Date;
function makeDate(year: number, month: number, day: number): Date;

// Implementation signature -- must be compatible with all overloads above,
// but is not itself visible to callers
function makeDate(yearOrTimestamp: number, month?: number, day?: number): Date {
  if (month !== undefined && day !== undefined) {
    return new Date(yearOrTimestamp, month - 1, day);
  }
  return new Date(yearOrTimestamp);
}

const fromTimestamp = makeDate(1700000000000);
const fromParts = makeDate(2026, 1, 15);
// makeDate(2026, 1);   // error -- no overload expects exactly 2 arguments
```

Overloads are a somewhat advanced tool — reach for a union parameter type
first, and use overloads only when the *number* or *combination* of
parameters genuinely changes shape between call patterns.

## `void` vs. omitted return

```typescript
function log(message: string): void {
  console.log(message);
  // returning a value here would still technically be allowed for `void`
  // (this quirk exists to support callback patterns), but don't rely on it --
  // treat void as "the return value is not meaningful."
}
```

## Cheat sheet

| Feature | Syntax | Notes |
|---------|--------|-------|
| Parameter type | `(x: number)` | Required unless annotated optional |
| Return type | `): number {` | TypeScript infers if omitted, but annotate public functions |
| Optional param | `(title?: string)` | Must follow required params; type includes `undefined` |
| Default param | `(exp: number = 2)` | Optional + supplies a fallback value |
| Rest param | `(...nums: number[])` | Collects trailing args into an array |
| Function type | `(a: number, b: number) => number` | Type of a callback / function value |
| Overloads | multiple signatures + one implementation | For calls that differ in argument shape |

## Exercise

Write a function `formatPrice(amount: number, currency: string = "USD"): string`
that returns a string like `"$42.00"` for USD or `"€42.00"` for EUR (default
to `"USD"` if not given). Then write a higher-order function
`retry(fn: () => boolean, attempts: number): boolean` that calls `fn` up to
`attempts` times, returning `true` as soon as `fn()` returns `true`, or
`false` if every attempt fails. Test it with a function that only "succeeds"
on its third call.
