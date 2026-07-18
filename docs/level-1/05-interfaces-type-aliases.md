# 05 · Interfaces & Type Aliases

Naming a "shape" of data is one of the most common things you'll do in
TypeScript. There are two tools for it: `interface` and `type`.

## Defining an interface

```typescript
interface User {
  id: number;
  name: string;
  email: string;
}

function printUser(user: User): void {
  console.log(`#${user.id}: ${user.name} <${user.email}>`);
}

const alice: User = { id: 1, name: "Alice", email: "alice@example.com" };
printUser(alice);   // #1: Alice <alice@example.com>

// const bad: User = { id: 2, name: "Bob" };
// error TS2741: Property 'email' is missing in type '{ id: number; name: string; }'
```

An object literal assigned to a `User`-typed variable must have exactly the
properties `User` declares (structural typing — more on this below).

## Optional and readonly properties

```typescript
interface Product {
  readonly sku: string;    // can be set once, never reassigned after creation
  name: string;
  discount?: number;       // optional -- may be omitted entirely
}

const item: Product = { sku: "ABC123", name: "Widget" };
// item.sku = "XYZ999";   // error TS2540: Cannot assign to 'sku' because it is a read-only property.

console.log(item.discount);   // undefined -- optional properties default to undefined when absent
```

## Type aliases

A `type` alias names *any* type — not just object shapes, but unions,
tuples, primitives, and more:

```typescript
type UserId = number;
type Point = { x: number; y: number };
type Status = "pending" | "active" | "closed";   // union of string literals

function findUser(id: UserId): void { /* ... */ }
function move(point: Point): void { /* ... */ }
function setStatus(status: Status): void { /* ... */ }

setStatus("active");     // fine
// setStatus("done");    // error -- "done" is not assignable to type 'Status'
```

## Interfaces vs. type aliases (a first look)

For plain object shapes, `interface` and `type` are nearly interchangeable:

```typescript
interface Animal {
  name: string;
}

type AnimalAlias = {
  name: string;
};
```

The most common rule of thumb: use `interface` for object shapes that might
need to be extended later (especially public APIs and classes), and `type`
for unions, tuples, or anything that isn't a plain object shape. The full
tradeoffs — declaration merging, `extends` vs. `&`, and when each one is the
better tool — get a dedicated deep dive in
[Level 2, Module 3](../level-2/03-interfaces-vs-types.md).

## Extending interfaces

```typescript
interface Shape {
  color: string;
}

interface Circle extends Shape {
  radius: number;
}

const c: Circle = { color: "red", radius: 5 };
console.log(c.color, c.radius);   // red 5
```

## Structural typing ("duck typing")

TypeScript compares shapes, not names — any object with the right properties
satisfies an interface, whether or not it was ever declared as one:

```typescript
interface Named {
  name: string;
}

function greet(entity: Named): string {
  return `Hello, ${entity.name}`;
}

// This plain object was never declared as `Named`, but it has a `name`
// property of type string -- that's enough to satisfy the interface.
const dog = { name: "Rex", breed: "Labrador" };
console.log(greet(dog));   // Hello, Rex
```

This is called **structural typing** (as opposed to the *nominal* typing used
by languages like Java or C#, where a type must be explicitly declared to
implement an interface). It's one of the most distinctive things about
TypeScript's type system.

## Nested object types

```typescript
interface Address {
  street: string;
  city: string;
  zip: string;
}

interface Customer {
  name: string;
  address: Address;
}

const customer: Customer = {
  name: "Jordan",
  address: { street: "1 Main St", city: "Springfield", zip: "00000" },
};

console.log(customer.address.city);   // Springfield
```

## Function types inside interfaces

```typescript
interface Calculator {
  label: string;
  compute: (a: number, b: number) => number;
}

const adder: Calculator = {
  label: "Adder",
  compute: (a, b) => a + b,
};

console.log(`${adder.label}: ${adder.compute(2, 3)}`);   // Adder: 5
```

## Cheat sheet

| Feature | Syntax | Notes |
|---------|--------|-------|
| Interface | `interface X { ... }` | Best for object shapes, can be extended/merged |
| Type alias | `type X = { ... }` | Can name unions, tuples, primitives too |
| Optional prop | `age?: number` | May be omitted; type includes `undefined` |
| Readonly prop | `readonly id: string` | Settable once, then immutable |
| `extends` | `interface B extends A` | Interface inheritance |
| Structural typing | -- | Shape match is enough, no explicit `implements` needed |

## Exercise

Define an interface `Book` with `readonly isbn: string`, `title: string`,
`author: string`, and an optional `rating?: number`. Write a function
`describeBook(book: Book): string` that returns a formatted string,
including the rating only if present. Then define a type alias
`LibraryStatus = "available" | "checked-out" | "lost"` and a function
`canBorrow(status: LibraryStatus): boolean` returning `true` only for
`"available"`.
