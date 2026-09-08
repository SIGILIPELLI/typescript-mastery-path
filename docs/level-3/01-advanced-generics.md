# 01 · Advanced Generics

Level 2 covered generic functions, classes, and `T extends keyof U`
constraints. This module goes further into the type-level features that
make TypeScript's generics genuinely powerful: conditional types that
branch on a type, `infer` to pull a type out of another type, mapped
types that transform every property of an object type, template literal
types, and variadic tuples. These are the tools behind library types like
`Awaited<T>`, `ReturnType<T>`, and `Parameters<T>` — after this module
you can read (and write) that kind of type yourself.

## Conditional types

A conditional type picks between two types based on a check, using the
same `T extends U ? X : Y` syntax as a ternary, but evaluated entirely at
the type level:

```typescript
type IsString<T> = T extends string ? "yes" : "no";

type A = IsString<string>;   // "yes"
type B = IsString<number>;   // "no"

const a: A = "yes";
const b: B = "no";
console.log(a, b);   // yes no
```

Nothing here runs at runtime — `IsString<T>` is resolved entirely by the
compiler while it checks your program, and by the time JavaScript exists
the conditional is gone.

## `infer` — extracting a type from within another type

`infer` introduces a new type variable inside the `extends` clause of a
conditional type, letting you "pull out" part of a type instead of just
branching on it:

```typescript
type ElementType<T> = T extends (infer U)[] ? U : T;

type NumEl = ElementType<number[]>;   // number
type StrEl = ElementType<string>;     // string (not an array, falls through to T)

const numEl: NumEl = 42;
const strEl: StrEl = "hi";
console.log(numEl, strEl);   // 42 hi
```

The same pattern unwraps a `Promise`, which is exactly how the built-in
`Awaited<T>` utility type works:

```typescript
type Unwrap<T> = T extends Promise<infer U> ? U : T;

type UnwrappedNum = Unwrap<Promise<number>>;   // number
type UnwrappedStr = Unwrap<string>;             // string

async function demo(): Promise<void> {
  const p: Promise<number> = Promise.resolve(10);
  const value: UnwrappedNum = await p;
  console.log("unwrapped:", value);
}
demo();   // unwrapped: 10
```

## A trap: conditional types distribute over unions by default

When `T` in `T extends U ? X : Y` is a **naked type parameter** and you
pass it a union, TypeScript checks the conditional against *each member
of the union separately* and unions the results back together:

```typescript
type ToArray<T> = T extends unknown ? T[] : never;
type StrOrNumArray = ToArray<string | number>;
// distributes to: ToArray<string> | ToArray<number>
// = string[] | number[]  -- NOT (string | number)[]

const distributed: StrOrNumArray = ["a", "b"];
console.log(distributed);   // [ 'a', 'b' ]
```

If you want the union treated as one single type instead, wrap both
sides in a tuple — a tuple isn't a "naked" type parameter, so
distribution doesn't kick in:

```typescript
type ToArrayNonDist<T> = [T] extends [unknown] ? T[] : never;
type CombinedArray = ToArrayNonDist<string | number>;   // (string | number)[]

const combined: CombinedArray = ["a", 1, "b"];
console.log(combined);   // [ 'a', 1, 'b' ]
```

This distinction is easy to miss and shows up as confusing type errors in
real code — if a conditional type's result looks wrong specifically when
you feed it a union, distribution is almost always the reason.

## Mapped types with modifiers

A mapped type transforms every property of an existing type. Level 2's
`Partial<T>`, `Readonly<T>`, and `Required<T>` are themselves mapped
types — here's how to write your own, including how to *remove* a
modifier with a leading `-`:

```typescript
interface Draft {
  title: string;
  body: string;
  tags: string[];
}

type ReadonlyDraft = { readonly [K in keyof Draft]: Draft[K] };

// Mutable<T> strips `readonly` back off with `-readonly`
type Mutable<T> = { -readonly [K in keyof T]: T[K] };

const ro: ReadonlyDraft = { title: "Hi", body: "...", tags: [] };
// ro.title = "no";
// error TS2540: Cannot assign to 'title' because it is a read-only property.

const mutableAgain: Mutable<ReadonlyDraft> = ro;
mutableAgain.title = "Edited";
console.log(mutableAgain.title);   // Edited
```

### Key remapping with `as`

Mapped types can also rename each property as they go, using `as` inside
the `[K in keyof T as ...]` clause — this is how you'd generate a set of
getter method names from a data interface:

```typescript
type Getters<T> = {
  [K in keyof T as `get${Capitalize<string & K>}`]: () => T[K];
};

type DraftGetters = Getters<Draft>;

const draftGetters: DraftGetters = {
  getTitle: () => "Hi",
  getBody: () => "...",
  getTags: () => [],
};
console.log(draftGetters.getTitle(), draftGetters.getTags());
// Hi []
```

## Template literal types

Template literal types build string literal unions the same way a
template string builds a runtime string, letting the type system
understand string *shapes*, not just exact values:

```typescript
type EventName = "click" | "hover" | "focus";
type HandlerName = `on${Capitalize<EventName>}`;
// "onClick" | "onHover" | "onFocus"

const handler: HandlerName = "onClick";
console.log(handler);   // onClick

function makeHandlerName(event: EventName): HandlerName {
  return `on${event.charAt(0).toUpperCase()}${event.slice(1)}` as HandlerName;
}
console.log(makeHandlerName("hover"));   // onHover
```

Note the `as HandlerName` at the end of `makeHandlerName` — the compiler
can't prove that string concatenation at runtime produces exactly one of
the literal union members, so an assertion is needed there, same as
casting any other computed value into a narrower type.

## Variadic tuple types

Type parameters can capture a variable-length tuple with `...T`, letting
generic functions describe "prepend one item" or "concatenate two
tuples" precisely, including the exact position of every element:

```typescript
type Concat<T extends unknown[], U extends unknown[]> = [...T, ...U];

type Combined = Concat<[string, number], [boolean]>;
// [string, number, boolean]

const combinedTuple: Combined = ["id", 1, true];
console.log(combinedTuple);   // [ 'id', 1, true ]

function prepend<T extends unknown[], V>(value: V, arr: [...T]): [V, ...T] {
  return [value, ...arr];
}

const withPrefix = prepend("start", [1, 2, 3] as const);
console.log(withPrefix);   // [ 'start', 1, 2, 3 ]
```

Without variadic tuples, a function like `prepend` could only be typed
as returning `(V | T[number])[]` — technically correct, but it throws
away the exact length and per-position types that the tuple version
keeps.

## How It Actually Works

Conditional types (`T extends U ? X : Y`) are evaluated by the checker as genuine type-level computation, not simple substitution — for each concrete `T` supplied, the checker performs a structural assignability check (is `T` assignable to `U`?) and picks branch `X` or `Y` accordingly, deferring the whole evaluation if `T` is still a generic (unresolved) type parameter at that point, which is why conditional types inside generic function bodies often show up as unresolved, unreduced types in tooling until the function is actually called with a concrete argument.

**Distribution** over unions (covered briefly in the utility-types lesson) is controlled by a specific syntactic rule: a conditional type distributes over a union *only* when the checked type is a "naked" type parameter — `T extends U ? X : Y` distributes when `T` is literally a type parameter reference, but `[T] extends [U] ? X : Y` (wrapping both sides in a tuple) suppresses distribution, forcing the whole union to be checked against `U` as one unit. This bracket trick exists specifically because sometimes you want the non-distributive behavior (e.g., checking `T extends never` against a whole union rather than each member), and there's no other syntax to opt out.

**`infer`** lets a conditional type extract a piece of a matched structure into a new type variable, resolved by the same structural pattern-matching the checker uses everywhere else — `T extends Promise<infer U> ? U : T` structurally matches `T` against the shape `Promise<?>`, and if it matches, binds whatever filled that position to `U`. Variadic tuple types (`[first: T, ...rest: R]`) extend this same structural matching to tuple *positions and lengths* — the checker can decompose and reconstruct tuple types by index the same way `infer` decomposes a generic type argument, which is the mechanism underlying utility types like `Parameters<F>` (destructuring a function's parameter tuple) and `ReturnType<F>` (both implemented as conditional types with `infer` in TypeScript's own lib).

## Cheat sheet

| Feature | Syntax | Use it for |
|---|---|---|
| Conditional type | `T extends U ? X : Y` | Branch on a type at compile time |
| Infer | `T extends (infer U)[] ? U : T` | Pull a type out of a wrapper type |
| Distribution | `T extends unknown ? T[] : never` | Applies the conditional per union member |
| Block distribution | `[T] extends [unknown] ? T[] : never` | Treats a union as one type |
| Mapped type | `{ [K in keyof T]: T[K] }` | Transform every property of `T` |
| Remove a modifier | `{ -readonly [K in keyof T]-?: T[K] }` | Strip `readonly`/`?` off a mapped type |
| Key remapping | `[K in keyof T as \`get${string & K}\`]` | Rename properties while mapping |
| Template literal type | `` `on${Capitalize<Event>}` `` | Build a string-literal union from parts |
| Variadic tuple | `[...T, ...U]` | Concatenate/prepend tuple types exactly |

## Exercise

Write a conditional type `Flatten<T>` that, given `T extends (infer U)[]
? U : T`, unwraps one level of array nesting (`Flatten<string[][]>` should
be `string[]`, and `Flatten<number>` should stay `number`). Then write a
mapped type `Nullable<T>` that makes every property of `T` allow `null`
in addition to its original type (`{ [K in keyof T]: T[K] | null }`), and
prove both work against a sample interface with a runtime value that
satisfies each resulting type.
