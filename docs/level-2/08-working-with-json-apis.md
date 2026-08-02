# 08 · Working with JSON/APIs

`fetch` and `JSON.parse` both return types that TypeScript cannot verify
against reality — `Promise<any>` and `any` respectively. Every "typed API
client" you'll ever build in TypeScript is really solving one problem:
turning that untrusted `any` into a type you can actually trust, as early
as possible.

## The starting point: `fetch` gives you `any`

```typescript
interface Todo {
  id: number;
  title: string;
  completed: boolean;
}

async function getTodo(id: number): Promise<Todo> {
  const response = await fetch(`https://example.com/todos/${id}`);
  const data = await response.json();   // `data` has type `any`
  return data;   // TypeScript trusts you completely here -- no check happens
}
```

This compiles cleanly, and that's exactly the problem: nothing here
verifies that the server actually sent an object shaped like `Todo`. If
the API changes, or returns an error object instead, `data` is silently
the wrong shape and the mistake surfaces later, somewhere confusing, as a
runtime crash (`Cannot read properties of undefined`) rather than a clear
type error.

## Step 1: at least assert the shape explicitly

```typescript
async function getTodoAsserted(id: number): Promise<Todo> {
  const response = await fetch(`https://example.com/todos/${id}`);
  const data = (await response.json()) as Todo;   // an assertion, not a check
  return data;
}
```

`as Todo` documents intent, but **`as` performs zero runtime checking** —
it just tells the compiler "trust me." It's strictly better than leaving
the type as `any` (at least the rest of your code gets checked against
`Todo` from here on), but it's not validation.

## Step 2: check the HTTP status before trusting the body

A very common bug: treating every response as success. `fetch` only
rejects its promise on a network failure — a 404 or 500 response still
resolves successfully:

```typescript
async function getTodoChecked(id: number): Promise<Todo> {
  const response = await fetch(`https://example.com/todos/${id}`);

  if (!response.ok) {
    // response.ok is false for any status outside 200-299
    throw new Error(`Request failed: ${response.status} ${response.statusText}`);
  }

  const data = (await response.json()) as Todo;
  return data;
}
```

## Step 3: actually validate the shape at runtime

A hand-written type guard (covered fully in
[Module 9](09-type-narrowing-guards.md)) checks the real shape of the
parsed JSON before you trust it as `Todo`:

```typescript
function isTodo(value: unknown): value is Todo {
  return (
    typeof value === "object" &&
    value !== null &&
    "id" in value &&
    typeof (value as Todo).id === "number" &&
    "title" in value &&
    typeof (value as Todo).title === "string" &&
    "completed" in value &&
    typeof (value as Todo).completed === "boolean"
  );
}

async function getTodoValidated(id: number): Promise<Todo> {
  const response = await fetch(`https://example.com/todos/${id}`);

  if (!response.ok) {
    throw new Error(`Request failed: ${response.status}`);
  }

  const data: unknown = await response.json();   // starts as `unknown`, not `any`

  if (!isTodo(data)) {
    throw new Error("Response did not match the expected Todo shape");
  }

  return data;   // narrowed to `Todo` -- genuinely checked, not just asserted
}
```

Typing the parsed result as `unknown` first (rather than `any`) is the
key habit: `unknown` forces every caller through a check like `isTodo`
before the value can be used as a `Todo`, whereas `any` would let it slip
through unchecked at every step.

## A generic, reusable typed fetch helper

Once you've validated at the boundary once, you can push the pattern into
a small generic helper — this is the same `T extends ...` idea from
[Module 2](02-generics.md), applied to networking:

```typescript
async function fetchJson<T>(
  url: string,
  validate: (data: unknown) => data is T
): Promise<T> {
  const response = await fetch(url);

  if (!response.ok) {
    throw new Error(`Request failed: ${response.status} ${response.statusText}`);
  }

  const data: unknown = await response.json();

  if (!validate(data)) {
    throw new Error(`Response from ${url} did not match the expected shape`);
  }

  return data;
}

// Usage:
async function loadTodo(id: number): Promise<Todo> {
  return fetchJson(`https://example.com/todos/${id}`, isTodo);
}
```

`fetchJson<T>` doesn't know or care what `T` is — it just requires a
matching validator function, and returns something callers can trust as
genuinely `T`, not just labeled `T`.

## Typing an array of results

```typescript
interface Post {
  id: number;
  userId: number;
  title: string;
}

function isPost(value: unknown): value is Post {
  return (
    typeof value === "object" &&
    value !== null &&
    "id" in value &&
    "userId" in value &&
    "title" in value
  );
}

function isPostArray(value: unknown): value is Post[] {
  return Array.isArray(value) && value.every(isPost);
}

async function loadPosts(): Promise<Post[]> {
  return fetchJson("https://example.com/posts", isPostArray);
}
```

`Array.isArray` narrows to `unknown[]`, and `.every(isPost)` re-checks
every element — without both parts, a response like `{ error: "oops" }`
(not even an array) or `[1, 2, 3]` (an array of the wrong element type)
would slip through undetected.

## Serializing outgoing JSON

The reverse direction — sending typed data as JSON — is simpler because
you control the shape, but still has a sharp edge worth knowing:

```typescript
interface NewPost {
  userId: number;
  title: string;
  body: string;
}

async function createPost(post: NewPost): Promise<void> {
  await fetch("https://example.com/posts", {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify(post),
  });
}

// A trap: JSON.stringify silently drops values it can't represent.
const withExtras = {
  userId: 1,
  title: "Hello",
  body: "World",
  createdAt: undefined,           // dropped entirely from the output
  logger: () => console.log("x"), // functions are dropped too
};

console.log(JSON.stringify(withExtras));
// {"userId":1,"title":"Hello","body":"World"}
// -- `createdAt` and `logger` are both gone, with no warning or error.
// TypeScript's structural typing won't catch this either, since
// `withExtras` still satisfies `NewPost` -- it just has EXTRA properties,
// which structural typing allows.
```

`JSON.stringify` quietly omits `undefined`, functions, and `symbol`
values. If a field disappearing from an API payload has ever confused
you, this is usually why — always log or test the actual serialized
string when debugging outgoing JSON, not just the object before
stringifying it.

## Cheat sheet

| Step | Code | Why |
|---|---|---|
| Raw fetch result | `await response.json()` | Type is `any` — trusts nothing |
| Assertion (no check) | `(await response.json()) as T` | Compiles, but performs zero runtime verification |
| Status check | `if (!response.ok) throw ...` | A 404/500 still resolves; only network failure rejects |
| Safe starting type | `const data: unknown = ...` | Forces a check before use, unlike `any` |
| Real validation | `function isT(x: unknown): x is T` | A type guard that actually inspects the shape |
| Reusable pattern | `fetchJson<T>(url, isT)` | Generic helper combining fetch + status check + validation |
| Array validation | `Array.isArray(x) && x.every(isT)` | Checks both "is an array" and "every element matches" |

## Exercise

Define an interface `Weather { temperatureC: number; conditions: string;
city: string }` and a type guard `isWeather(value: unknown): value is
Weather`. Write `fetchWeather(city: string): Promise<Weather>` using the
generic `fetchJson<T>` helper from this module against a URL you
construct from the `city` parameter. Then deliberately break the
happy path by validating a hand-written object that's missing
`conditions` and confirm `isWeather` correctly returns `false` for it.
This exercise is a warm-up for the [Level 2 project](10-project-weather-dashboard.md),
which builds a full weather dashboard using exactly this pattern.
