# 03 · Interfaces vs Type Aliases Deep Dive

[Level 1](../level-1/05-interfaces-type-aliases.md) gave you the quick
rule of thumb: `interface` for object shapes, `type` for everything else.
That's good enough for most days, but real codebases eventually hit the
cases where the choice actually matters — declaration merging, how each
one combines with others, and where one can express something the other
flatly cannot.

## The 90% overlap

For a plain object shape, both forms produce equivalent, mutually
assignable types:

```typescript
interface UserI {
  id: number;
  name: string;
}

type UserT = {
  id: number;
  name: string;
};

function printUser(u: UserI): void {
  console.log(`#${u.id} ${u.name}`);
}

const asType: UserT = { id: 1, name: "Amy" };
printUser(asType);   // fine -- structural typing doesn't care which
                      // declaration form produced the shape
// #1 Amy
```

Both support optional (`?`) and `readonly` properties, function-typed
properties, and generics. The differences show up around the edges.

## Difference 1: declaration merging

Interfaces with the same name in the same scope **merge** automatically;
type aliases with the same name are a compile error:

```typescript
interface Window {
  title: string;
}

interface Window {
  height: number;
}

// Merged automatically into: { title: string; height: number }
const w: Window = { title: "Main", height: 600 };
console.log(w.title, w.height);   // Main 600

// type Config = { title: string };
// type Config = { height: number };
// error TS2300: Duplicate identifier 'Config'.
```

This isn't a party trick — it's how ambient type declarations extend
third-party libraries (e.g. adding custom properties to Express's
`Request` type) without editing the library's source. It's also a trap
when unintended: an interface named the same thing as one in a library
you imported can silently merge rather than error, quietly reshaping a
type you didn't mean to touch.

## Difference 2: `extends` vs `&`

Interfaces extend other interfaces; type aliases combine with `&`. They
look similar but behave differently on conflicting members:

```typescript
interface Base {
  id: string;
}

interface Derived extends Base {
  id: string;   // fine -- same type as Base.id, this is allowed
  extra: number;
}

// interface Conflicting extends Base {
//   id: number;   // error TS2430: Interface 'Conflicting' incorrectly
//                 // extends interface 'Base'. Types of property 'id'
//                 // are incompatible.
// }
```

An interface `extends` clause **checks compatibility immediately** and
errors right at the declaration if the shapes conflict. An intersection
type, on the other hand, just merges — and if the merge is impossible, it
resolves the conflicting property to `never` instead of raising an error
at the declaration site:

```typescript
type BaseT = { id: string };
type ConflictingT = BaseT & { id: number };
// No error here! ConflictingT.id has type `string & number`, which is `never`.

// const bad: ConflictingT = { id: "x" };
// error TS2322: Type 'string' is not assignable to type 'never'.
// The error only surfaces later, when you actually try to use the type --
// which can make the root cause harder to track down.
```

This is a genuine trap: prefer `interface extends` over `&` when you
expect the shapes might conflict, specifically because it fails fast at
the point of the mistake rather than somewhere downstream.

## Difference 3: only type aliases name non-object types

`interface` can only describe object-like shapes (including function and
constructor signatures). `type` can name anything:

```typescript
type ID = string | number;                 // interface cannot do this
type Coordinates = [number, number];       // or this (tuple)
type Handler = (event: string) => void;    // interfaces CAN do this one too,
                                            // but the type-alias form reads
                                            // more naturally for a bare function type
type Nullable<T> = T | null;               // generic alias over a union

const id: ID = 42;
const point: Coordinates = [10, 20];
const onClick: Handler = (event) => console.log(`Clicked: ${event}`);
let maybe: Nullable<string> = null;

console.log(id, point, onClick.name === "" ? "(anonymous)" : onClick.name);
onClick("button");   // Clicked: button
maybe = "now set";
console.log(maybe);   // now set
```

If what you're naming isn't fundamentally an object shape — a union, a
tuple, a mapped type, a conditional type — reach for `type`. There's no
equivalent `interface` syntax for a bare union or tuple.

## Difference 4: interfaces support implicit merging with classes

A `class` can `implement` either form, but interfaces play a special role
when you want a type that's automatically satisfied by any object with
the right shape *and* documents an intended contract clearly:

```typescript
interface Serializable {
  serialize(): string;
}

class Invoice implements Serializable {
  constructor(private amount: number) {}

  serialize(): string {
    return JSON.stringify({ amount: this.amount });
  }
}

const inv = new Invoice(500);
console.log(inv.serialize());   // {"amount":500}
```

`type` works identically here too (`type Serializable = { serialize():
string }` and `implements Serializable` compiles the same way) — this is
really a style convention, not a functional difference. Public library
APIs and anything meant to be `implements`-ed by consumers lean toward
`interface` because merging lets consumers extend it later.

## Practical rule of thumb

| Situation | Prefer |
|---|---|
| Plain object shape, especially a public API | `interface` |
| Might need to be extended by consumers later | `interface` |
| Union, tuple, primitive alias, or mapped/conditional type | `type` |
| You need declaration merging (extending a library's types) | `interface` |
| Combining several shapes where conflicts should fail loudly | `interface extends` |
| One-off, internal, "just needs a name for this specific shape" | either — pick a team convention and stay consistent |

## How It Actually Works

The checker treats `interface` and object-shape `type` aliases as producing the same underlying structural type once resolved — assignability comparisons don't care which keyword declared a shape. The real differences are in *how and when* the compiler resolves each declaration. An `interface` is resolved **lazily and incrementally**: the checker can reference an interface's own members while still processing its declaration (self-reference), and declaration merging (covered in an earlier lesson) works because each `interface X { }` block is treated as a partial contribution to one accumulated member list, only flattened into a final shape when something actually needs to check against `X`.

A `type` alias for an object shape, by contrast, is resolved **eagerly** at the point of declaration into a fixed type — no merging, and a self-referencing type alias (`type Tree = { children: Tree[] }`) only works because object/array member positions are lazily evaluated internally by the checker, not because type aliases support recursion in general; a *non-object* self-referential alias (`type Bad = Bad | string`) is rejected as circular precisely because there's no member boundary to defer evaluation across.

This resolution difference is also why large unions of `interface extends` chains can be measurably slower to check than equivalent `type` intersections in some compiler versions — merged-and-then-flattened interface resolution vs. eagerly-computed intersection types exercise different code paths in the checker's caching, though this has narrowed across TypeScript releases and isn't a reliable rule to design around.

Neither form has any runtime representation — both compile to nothing, and both are checked through the same structural, member-by-member comparison; the choice between them is entirely about which authoring-time behaviors (mergeability, union/mapped-type operations only `type` supports) you need, not about differing runtime or type-safety guarantees.

## Cheat sheet

| Feature | `interface` | `type` |
|---|---|---|
| Object shapes | Yes | Yes |
| Unions / tuples / primitives | No | Yes |
| Declaration merging | Yes (same name merges) | No (duplicate name errors) |
| Combine with conflict detection | `extends` (errors early) | `&` (resolves to `never` silently) |
| Generic | Yes | Yes |
| `implements` in a class | Yes | Yes |

## Exercise

Declare an interface `Vehicle` with `make: string` and `model: string`.
Separately, in the same file, declare a *second* `interface Vehicle` that
adds `year: number` — confirm they merge into one three-property type.
Then write two type aliases, `Success = { status: "success"; data: string
}` and `Failure = { status: "failure"; error: string }`, and a union
`Outcome = Success | Failure` — this is something `interface` alone could
not express directly. Write a function `report(outcome: Outcome): string`
that narrows on `status` and returns the right message for each case.
