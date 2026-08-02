# 02 · Generics

Generics let you write a function, class, or type once and have it work
correctly for many different types — without falling back to `any` and
losing type safety. If you've ever wondered how `Array<T>` or `Promise<T>`
work for every possible type without a thousand copy-pasted definitions,
this is how.

## The problem generics solve

Without generics, you either duplicate code per type or give up type
safety with `any`:

```typescript
function firstStringElement(arr: string[]): string {
  return arr[0];
}

function firstNumberElement(arr: number[]): number {
  return arr[0];
}

// Or the "give up" version:
function firstAny(arr: any[]): any {
  return arr[0];
}

const oops = firstAny([1, 2, 3]);
oops.toUpperCase();   // no error at compile time -- but crashes at runtime:
// TypeError: oops.toUpperCase is not a function
```

`any` disables type checking entirely for that value — it's the escape
hatch, not a type. Every `any` is a place a real bug can hide until
runtime.

## Generic functions

A generic function introduces a **type parameter** (conventionally `T`)
that stands in for "whatever type the caller passes":

```typescript
function firstElement<T>(arr: T[]): T {
  return arr[0];
}

const num = firstElement([1, 2, 3]);          // inferred as number
const str = firstElement(["a", "b", "c"]);    // inferred as string

console.log(num.toFixed(1));      // 1.0
console.log(str.toUpperCase());   // A

// str.toFixed(1);
// error TS2339: Property 'toFixed' does not exist on type 'string'.
```

TypeScript infers `T` from the argument you pass — you rarely need to
write it explicitly, though you can: `firstElement<number>([1, 2, 3])`.

### Multiple type parameters

```typescript
function pair<A, B>(first: A, second: B): [A, B] {
  return [first, second];
}

const p = pair("age", 30);
console.log(p);   // [ 'age', 30 ]
// p is typed as [string, number] -- a tuple, not just (string | number)[]
```

## Generic constraints

Sometimes `T` needs to guarantee it has certain properties. `extends`
constrains what types are allowed:

```typescript
interface HasLength {
  length: number;
}

function logLength<T extends HasLength>(value: T): T {
  console.log(`Length: ${value.length}`);
  return value;
}

logLength("hello");        // Length: 5
logLength([1, 2, 3]);      // Length: 3
logLength({ length: 10, unit: "cm" });   // Length: 10

// logLength(42);
// error TS2345: Argument of type 'number' is not assignable to
// parameter of type 'HasLength'.
```

Without the constraint, `value.length` wouldn't compile at all — plain
`T` guarantees nothing about its shape.

### Constraining to a key of another type

A very common pattern: a function that looks up a property, constrained
so the key must actually exist on the object:

```typescript
function getProperty<T, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key];
}

const user = { id: 1, name: "Nina", active: true };

console.log(getProperty(user, "name"));     // Nina
console.log(getProperty(user, "active"));   // true

// getProperty(user, "email");
// error TS2345: Argument of type '"email"' is not assignable to
// parameter of type '"id" | "name" | "active"'.
```

`K extends keyof T` is worth memorizing — it's how libraries type-safely
model "pick one field off this object" without `any` or string literals
that could typo silently.

## Generic classes

Classes can be generic too, letting one implementation serve many element
types:

```typescript
class Box<T> {
  private contents: T;

  constructor(value: T) {
    this.contents = value;
  }

  get(): T {
    return this.contents;
  }

  set(value: T): void {
    this.contents = value;
  }
}

const numberBox = new Box<number>(42);
console.log(numberBox.get());   // 42

const stringBox = new Box("hello");   // T inferred as string
stringBox.set("world");
console.log(stringBox.get());   // world

// numberBox.set("nope");
// error TS2345: Argument of type 'string' is not assignable to
// parameter of type 'number'.
```

### A generic stack

```typescript
class Stack<T> {
  private items: T[] = [];

  push(item: T): void {
    this.items.push(item);
  }

  pop(): T | undefined {
    return this.items.pop();
  }

  peek(): T | undefined {
    return this.items[this.items.length - 1];
  }

  get size(): number {
    return this.items.length;
  }
}

const stack = new Stack<string>();
stack.push("first");
stack.push("second");
console.log(stack.peek());   // second
console.log(stack.size);     // 2
console.log(stack.pop());    // second
console.log(stack.size);     // 1
```

Note `pop(): T | undefined` — it's tempting to write just `T`, but that
would be a lie: popping an empty stack really does return `undefined` at
runtime, and the type should say so.

## Default type parameters

Type parameters can have defaults, just like function parameters:

```typescript
interface ApiResponse<T = unknown> {
  data: T;
  success: boolean;
}

const withDefault: ApiResponse = { data: "anything", success: true };
const typed: ApiResponse<number> = { data: 42, success: true };

console.log(withDefault.data, typed.data);   // anything 42
```

Prefer defaulting to `unknown` rather than `any` — `unknown` still forces
callers to narrow before using the value, while `any` waives all checking.

## Generic type aliases

```typescript
type Result<T, E = Error> =
  | { ok: true; value: T }
  | { ok: false; error: E };

function divide(a: number, b: number): Result<number, string> {
  if (b === 0) {
    return { ok: false, error: "Division by zero" };
  }
  return { ok: true, value: a / b };
}

const r1 = divide(10, 2);
const r2 = divide(10, 0);

if (r1.ok) {
  console.log(r1.value);   // 5
}
if (!r2.ok) {
  console.log(r2.error);   // Division by zero
}
```

This `Result<T, E>` pattern shows up constantly once you start typing
things like API calls and file parsing — see it again in
[Module 6](06-async-await-types.md) and the
[weather dashboard project](10-project-weather-dashboard.md).

## A trap: generics don't validate anything at runtime

Generics are a **compile-time-only** construct — they're erased entirely
when TypeScript compiles to JavaScript. A generic function can't check at
runtime that the value it receives actually matches `T`:

```typescript
function trustinglyCast<T>(value: unknown): T {
  return value as T;   // no runtime check happens here at all
}

interface Cat { meow(): void }

const notActuallyACat = trustinglyCast<Cat>({ bark: () => console.log("Woof") });
// TypeScript believes notActuallyACat is a Cat -- it compiles fine.
// notActuallyACat.meow();
// Runtime error: notActuallyACat.meow is not a function
```

Generics constrain what the *type checker* accepts; they do nothing to
verify data that genuinely arrives from outside your program (JSON
payloads, user input, files). That's the job of runtime validation,
covered in [Module 8](08-working-with-json-apis.md).

## Cheat sheet

| Syntax | Meaning |
|---|---|
| `function f<T>(x: T): T` | Generic function, `T` inferred from the call site |
| `class Box<T> { ... }` | Generic class, instantiated as `new Box<number>(...)` |
| `T extends HasLength` | Constrain `T` to types with a `length` property |
| `K extends keyof T` | Constrain `K` to an actual key of `T` |
| `Box<T = unknown>` | Default type parameter if none is given |
| `T[]` / `Array<T>` | Equivalent generic array syntaxes |

## Exercise

Write a generic function `pluck<T, K extends keyof T>(items: T[], key: K):
T[K][]` that extracts a single field from every object in an array (e.g.
`pluck(users, "name")` returns `string[]`). Then write a generic class
`Queue<T>` with `enqueue(item: T): void`, `dequeue(): T | undefined`, and
a `size` getter, and prove it works with both a `Queue<number>` and a
`Queue<string>` in the same file.
