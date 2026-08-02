# 01 · Union & Intersection Types

Level 1 introduced union types as string-literal switches (`"pending" |
"active" | "closed"`). This module goes deeper: how unions and
intersections actually combine types, the "widest common ground" trap that
catches beginners, and the discriminated union pattern that professional
TypeScript code leans on constantly.

## Union types: "this OR that"

A union type describes a value that could be one of several types:

```typescript
function formatId(id: number | string): string {
  return `ID-${id}`;
}

console.log(formatId(42));      // ID-42
console.log(formatId("42"));    // ID-42
// formatId(true);
// error TS2345: Argument of type 'boolean' is not assignable to
// parameter of type 'string | number'.
```

Inside a function that receives a union, you only get access to the
members and methods common to **every** type in the union — TypeScript
won't let you assume it's narrowed down yet:

```typescript
function printLength(value: string | number): void {
  // console.log(value.length);
  // error TS2339: Property 'length' does not exist on type 'number'.

  if (typeof value === "string") {
    console.log(value.length);   // fine -- narrowed to string here
  } else {
    console.log(value.toFixed(0));   // fine -- narrowed to number here
  }
}

printLength("hello");   // 5
printLength(3.14159);   // 3
```

That `typeof` check is a **type guard** — narrowing gets a full module of
its own in [Module 9](09-type-narrowing-guards.md).

## Unions of object types

Unions aren't limited to primitives. This is where they get genuinely
useful for modeling real data:

```typescript
interface Circle {
  kind: "circle";
  radius: number;
}

interface Square {
  kind: "square";
  side: number;
}

type Shape = Circle | Square;

function area(shape: Shape): number {
  if (shape.kind === "circle") {
    return Math.PI * shape.radius ** 2;
  }
  return shape.side ** 2;
}

console.log(area({ kind: "circle", radius: 2 }).toFixed(2));   // 12.57
console.log(area({ kind: "square", side: 3 }));                 // 9
```

The shared `kind` literal property is called a **discriminant**. Because
every member of the union has it, and each member gives it a different
literal value, TypeScript can narrow the whole object just by checking
that one field. This pattern is called a **discriminated union** (or
"tagged union"), and it's one of the most powerful modeling tools in the
language — reach for it any time you have "one of several kinds of thing."

### Exhaustiveness checking with `never`

The discriminated union pattern pairs with a compiler trick that catches
bugs when you add a new variant and forget to handle it somewhere:

```typescript
interface Triangle {
  kind: "triangle";
  base: number;
  height: number;
}

type Shape2 = Circle | Square | Triangle;

function area2(shape: Shape2): number {
  switch (shape.kind) {
    case "circle":
      return Math.PI * shape.radius ** 2;
    case "square":
      return shape.side ** 2;
    case "triangle":
      return (shape.base * shape.height) / 2;
    default:
      // If every case above is handled, `shape` has type `never` here.
      // If you add a 4th shape and forget a case, this line fails to
      // compile -- the compiler catches the gap for you.
      const exhaustive: never = shape;
      throw new Error(`Unhandled shape: ${JSON.stringify(exhaustive)}`);
  }
}

console.log(area2({ kind: "triangle", base: 4, height: 5 }));   // 10
```

## Intersection types: "this AND that"

An intersection combines multiple types into one that must satisfy all of
them at once, using `&`:

```typescript
interface Named {
  name: string;
}

interface Aged {
  age: number;
}

type Person = Named & Aged;

const dev: Person = { name: "Priya", age: 29 };
console.log(`${dev.name} is ${dev.age}`);   // Priya is 29

// const incomplete: Person = { name: "Sam" };
// error TS2739: Property 'age' is missing in type '{ name: string; }'
```

Intersections are especially handy for composing smaller, reusable pieces
into a bigger shape — think mixing in "timestamped", "identifiable", or
"paginated" fragments:

```typescript
interface Timestamped {
  createdAt: Date;
  updatedAt: Date;
}

interface Identifiable {
  id: string;
}

type PostComment = Identifiable & Timestamped & {
  text: string;
};

const comment: PostComment = {
  id: "c1",
  text: "Nice post!",
  createdAt: new Date("2024-01-01"),
  updatedAt: new Date("2024-01-01"),
};

console.log(comment.id, comment.text);   // c1 Nice post!
```

!!! warning "Watch your naming"
    `Comment` looks like a harmless name, but it collides with the global
    DOM `Comment` interface that ships in TypeScript's default library
    (`lib.dom.d.ts`), which is included by default even in Node projects
    unless you override `lib` in `tsconfig.json`. The fix here isn't a
    TypeScript feature — it's a habit: avoid reusing common DOM/global
    names (`Comment`, `Event`, `Location`, `History`) for your own domain
    types.

## The primitive-intersection trap

A common mistake: expecting `&` on primitive types to "combine" them the
way it combines object shapes. It doesn't — it computes types that could
satisfy *both* constraints simultaneously, which for incompatible
primitives is impossible:

```typescript
// type Weird = string & number;
// Weird resolves to `never` -- no value can be both a string and a number
// at once. TypeScript will happily let you declare this type; it's only
// a problem once you try to produce a value of it.

// function impossible(): Weird {
//   return "oops";
// }
// error TS2322: Type 'string' is not assignable to type 'never'.
```

Remember the rule: unions widen ("could be either"), intersections narrow
("must satisfy both"). For two compatible object shapes, narrowing means
"has every property from both." For two unrelated primitives, narrowing
collapses all the way down to `never`.

## Unions and excess property checks

Object literals assigned directly to a union type still go through
TypeScript's excess property check, which can produce a confusing error
if a literal doesn't match any single member cleanly:

```typescript
type Contact = { email: string } | { phone: string };

const c1: Contact = { email: "a@b.com" };    // fine -- matches first member
const c2: Contact = { phone: "555-0100" };   // fine -- matches second member

// const c3: Contact = { email: "a@b.com", phone: "555-0100" };
// error TS2322: Object literal may only specify known properties, and
// 'phone' does not exist in type '{ email: string; }'.
// (Even though the object satisfies the *second* member alone, a fresh
// object literal is checked against a "best match" and flagged for
// extra properties not on that particular member.)

const viaVariable = { email: "a@b.com", phone: "555-0100" };
const c4: Contact = viaVariable;
// Fine -- assigning via a variable skips the excess property check
// entirely, since the checker no longer has a fresh literal to inspect.
```

This is a genuine TypeScript trap: the excess property check only fires on
*fresh object literals*, not on values passed through a variable first.
Don't rely on it as validation — it's a linting convenience, not a runtime
guarantee.

## Cheat sheet

| Concept | Syntax | Meaning |
|---|---|---|
| Union | `A \| B` | Value is one of these types |
| Intersection | `A & B` | Value satisfies all of these types |
| Discriminant | shared literal field, e.g. `kind: "circle"` | Lets TS narrow a union automatically |
| Exhaustiveness check | `const x: never = value` in a `default` | Compiler error if a case is missing |
| Primitive intersection | `string & number` | Resolves to `never` -- no value can satisfy both |
| Excess property check | only on fresh object literals | Assign via a variable to bypass it |

## Exercise

Model a notification system with a discriminated union `Notification =
EmailNotification | SmsNotification | PushNotification`, each with a
`type` discriminant literal and its own extra fields (e.g. `subject` and
`body` for email, `phoneNumber` and `message` for SMS, `title` and
`deviceId` for push). Write a function `describe(n: Notification):
string` using a `switch` on `type` that returns a one-line summary per
kind, and add a `default` branch with the `never` exhaustiveness trick so
that adding a fourth notification type without updating `describe` is a
compile error.
