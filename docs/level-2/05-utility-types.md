# 05 · Utility Types

TypeScript ships a set of built-in generic types that transform existing
types into new ones — making a type's fields optional, picking a subset
of fields, building a dictionary type, and more. They save you from
hand-writing (and forgetting to update) near-duplicate interfaces.

## Setup type used throughout this module

```typescript
interface Article {
  id: string;
  title: string;
  body: string;
  tags: string[];
  published: boolean;
}
```

## `Partial<T>` — every property optional

Useful for "update" functions where the caller only sends the fields
that changed:

```typescript
function updateArticle(id: string, changes: Partial<Article>): void {
  console.log(`Updating ${id} with`, changes);
}

updateArticle("a1", { title: "New Title" });          // fine -- other fields omitted
updateArticle("a2", { published: true, tags: [] });   // fine -- any subset

// The plain Article type would reject both of these calls:
// updateArticle("a3", {} as Article);
// (compiles, but only because of the `as` cast bypassing the check --
// without it, an empty object literal is missing every required field)
```

## `Required<T>` — the opposite of `Partial`

Forces every optional property to be present:

```typescript
interface Options {
  timeout?: number;
  retries?: number;
}

function run(options: Required<Options>): void {
  console.log(`timeout=${options.timeout} retries=${options.retries}`);
}

run({ timeout: 1000, retries: 3 });   // fine
// run({ timeout: 1000 });
// error TS2345: Property 'retries' is missing in type '{ timeout: number; }'
// but required in type 'Required<Options>'.
```

## `Readonly<T>` — freeze every property at the type level

```typescript
const draft: Readonly<Article> = {
  id: "a1",
  title: "Draft",
  body: "...",
  tags: [],
  published: false,
};

// draft.title = "Renamed";
// error TS2540: Cannot assign to 'title' because it is a read-only property.
```

Like `readonly` on a single property, this is a compile-time guarantee
only — it doesn't deep-freeze the object at runtime (`draft.tags.push(...)`
would still compile and run, since `tags` is `readonly` as a *reference*,
not as a deeply immutable array).

## `Pick<T, Keys>` — a subset of properties

```typescript
type ArticlePreview = Pick<Article, "id" | "title" | "tags">;

const preview: ArticlePreview = {
  id: "a1",
  title: "Intro to TypeScript",
  tags: ["typescript", "tutorial"],
};

console.log(preview.title);   // Intro to TypeScript
// preview.body;
// error TS2339: Property 'body' does not exist on type 'ArticlePreview'.
```

## `Omit<T, Keys>` — everything except some properties

The mirror image of `Pick` — useful when a "create" payload has all the
domain fields except a server-generated `id`:

```typescript
type NewArticle = Omit<Article, "id">;

function createArticle(data: NewArticle): Article {
  return { id: crypto.randomUUID(), ...data };
}

const created = createArticle({
  title: "Utility Types",
  body: "...",
  tags: ["typescript"],
  published: false,
});

console.log(created.id.length > 0, created.title);   // true Utility Types
```

## `Record<Keys, Type>` — a typed dictionary

```typescript
type Status = "draft" | "review" | "published";

const statusLabels: Record<Status, string> = {
  draft: "Draft",
  review: "In Review",
  published: "Live",
};

console.log(statusLabels.review);   // In Review

// const incomplete: Record<Status, string> = { draft: "Draft", review: "In Review" };
// error TS2741: Property 'published' is missing in type
// '{ draft: string; review: string; }' but required in type 'Record<Status, string>'.
```

Unlike a plain `{ [key: string]: string }` index signature, `Record` with
a union of literal keys forces you to provide **every** key — it's both a
dictionary type and an exhaustiveness check in one.

## `Exclude<T, U>` and `Extract<T, U>` — filtering unions

```typescript
type AllStatus = "draft" | "review" | "published" | "archived";

type ActiveStatus = Exclude<AllStatus, "archived">;
// "draft" | "review" | "published"

type FinalStatus = Extract<AllStatus, "published" | "archived">;
// "published" | "archived"

const active: ActiveStatus = "review";
const final: FinalStatus = "archived";
console.log(active, final);   // review archived
```

`Exclude` removes union members that match; `Extract` keeps only the
members that match. They're easy to mix up — a mnemonic: **Extract**
keeps what matches, **Exclude** throws out what matches.

## `ReturnType<T>` and `Parameters<T>` — types from functions

Handy when you don't own the function's declaration but need its types:

```typescript
function fetchArticle(id: string, includeDraft: boolean) {
  return { id, title: "Example", includeDraft };
}

type FetchResult = ReturnType<typeof fetchArticle>;
type FetchArgs = Parameters<typeof fetchArticle>;

const result: FetchResult = { id: "a1", title: "Example", includeDraft: false };
const args: FetchArgs = ["a1", true];

console.log(result, args);
// { id: 'a1', title: 'Example', includeDraft: false } [ 'a1', true ]
```

Note `typeof fetchArticle` — that's using the *value* `fetchArticle` in a
type position to get its function type, which `ReturnType`/`Parameters`
then unwrap. This combination is extremely common in real code that
derives types from existing functions rather than duplicating them.

## Combining utility types

Utility types compose naturally, since each one just takes and returns a
type:

```typescript
type ArticleFormState = Partial<Pick<Article, "title" | "body" | "tags">>;

const formState: ArticleFormState = { title: "In progress..." };
console.log(formState);   // { title: 'In progress...' }
```

## A trap: `Partial` doesn't mean "safe to skip validation"

`Partial<T>` is a compile-time convenience for the shape of *your own
code's* internal calls. It says nothing about data coming from outside
the program:

```typescript
function applyUpdate(changes: Partial<Article>): void {
  // If `changes` actually came from `JSON.parse(request.body)`, TypeScript
  // has no way to verify at runtime that it truly matches Partial<Article>
  // -- a malformed request could hand you a `tags: "not-an-array"` and
  // TypeScript would never know, because JSON.parse returns `any`.
  console.log(changes);
}
```

Runtime validation of untrusted data is covered in
[Module 8](08-working-with-json-apis.md) — utility types alone are never
a substitute for it.

## How It Actually Works

Utility types like `Partial<T>`, `Required<T>`, `Readonly<T>`, and `Pick<T, K>` are not compiler magic — they're ordinary generic **mapped types**, defined in TypeScript's own standard library `.d.ts` and expressible with the same syntax available to you. `Partial<T>` is literally defined as `{ [P in keyof T]?: T[P] }`: `keyof T` produces a union of `T`'s property-name literal types, `[P in keyof T]` iterates that union as a mapped-type index, and `?` adds the optional modifier to every mapped property. Because it's just a mapped type, the checker evaluates `Partial<User>` by actually performing this iteration over `User`'s real members at the point of use — it's a genuine type-level computation, not a lookup table of special cases.

`Pick<T, K extends keyof T>` (`{ [P in K]: T[P] }`) and `Omit<T, K>` (defined via `Pick<T, Exclude<keyof T, K>>`) show how utility types compose from smaller primitives: `Exclude<T, U>` itself is a **conditional type** (`T extends U ? never : T`) that, when `T` is a union, is applied to *each member of the union separately* — this is **distributive conditional type** behavior, and it's why `Exclude<"a" | "b", "a">` correctly narrows to `"b"` rather than evaluating the condition once against the whole union: the compiler distributes the conditional across each union member, checks each independently, and unions the survivors back together.

All of this resolution happens entirely during type checking; none of it produces runtime code. `Readonly<T>` is a particularly sharp example of the type-erasure boundary — it adds a `readonly` modifier the checker enforces on write-*expressions* in your source, but the compiled object is a completely ordinary, mutable JS object; `Object.freeze` is the only way to get an actual runtime immutability guarantee, and the two are unrelated — `Readonly<T>` alone won't stop `Object.assign(obj, ...)`'s internals or a caller who genuinely doesn't type-check their code from mutating the object at runtime.

## Cheat sheet

| Utility | Signature | Does |
|---|---|---|
| `Partial<T>` | `Partial<T>` | Every property optional |
| `Required<T>` | `Required<T>` | Every property required |
| `Readonly<T>` | `Readonly<T>` | Every property `readonly` (shallow, compile-time only) |
| `Pick<T, K>` | `Pick<T, "a" \| "b">` | Keep only listed properties |
| `Omit<T, K>` | `Omit<T, "a">` | Keep everything except listed properties |
| `Record<K, V>` | `Record<"a" \| "b", V>` | Dictionary keyed by a union, all keys required |
| `Exclude<T, U>` | `Exclude<T, U>` | Union members NOT assignable to `U` |
| `Extract<T, U>` | `Extract<T, U>` | Union members assignable to `U` |
| `ReturnType<T>` | `ReturnType<typeof fn>` | The return type of a function |
| `Parameters<T>` | `Parameters<typeof fn>` | A tuple of a function's parameter types |

## Exercise

Starting from an interface `Product { id: string; name: string; price:
number; stock: number; category: string }`, define: `ProductSummary`
using `Pick` with only `id`, `name`, and `price`; `ProductDraft` using
`Omit` to drop `id` (server-generated); `ProductUpdate` as
`Partial<Omit<Product, "id">>` for a PATCH-style update function; and a
`Record<string, Product[]>` called `productsByCategory` that groups a
small hardcoded array of products by their `category` field.
