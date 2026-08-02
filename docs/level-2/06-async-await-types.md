# 06 · Async/Await with Types

Every `async` function returns a `Promise`, and TypeScript tracks exactly
what that promise resolves to. Once you know how `Promise<T>` composes
with generics, `async`/`await`, and error handling, typed asynchronous
code stops feeling different from typed synchronous code.

## The type of an `async` function

```typescript
async function getGreeting(): Promise<string> {
  return "Hello!";
}

// Note: you write the RESOLVED type, not `Promise<Promise<string>>` --
// TypeScript automatically wraps a plain `string` return in a Promise.

getGreeting().then((message) => console.log(message));   // Hello!

async function main1(): Promise<void> {
  const message = await getGreeting();   // `message` is `string`, already unwrapped
  console.log(message.toUpperCase());    // HELLO!
}

main1();
```

Inside an `async` function, `await` unwraps a `Promise<T>` down to `T`.
Outside one, you work with the `Promise<T>` itself via `.then()`.

## Typing a function that wraps a real async operation

```typescript
function delay(ms: number): Promise<void> {
  return new Promise((resolve) => setTimeout(resolve, ms));
}

async function fetchUserName(id: number): Promise<string> {
  await delay(50);   // simulate network latency
  const names: Record<number, string> = { 1: "Ada", 2: "Grace" };
  return names[id] ?? "Unknown";
}

async function main2(): Promise<void> {
  const name = await fetchUserName(1);
  console.log(`User: ${name}`);   // User: Ada
}

main2();
```

## Awaiting multiple promises with `Promise.all`

`Promise.all` preserves the individual types of each promise in a tuple,
not a generic array:

```typescript
async function fetchThree(): Promise<void> {
  const [name, age, active] = await Promise.all([
    Promise.resolve("Nadia"),
    Promise.resolve(31),
    Promise.resolve(true),
  ]);

  // name: string, age: number, active: boolean -- each correctly typed,
  // not widened to `(string | number | boolean)[]`
  console.log(`${name}, ${age}, active=${active}`);   // Nadia, 31, active=true
}

fetchThree();
```

## Error handling: `try`/`catch` and the `unknown` catch type

Since TypeScript 4.4, values caught in a `catch` block are typed
`unknown` by default (previously `any`) — a deliberate safety
improvement, since JavaScript allows `throw`ing literally anything, not
just `Error` objects:

```typescript
async function riskyOperation(shouldFail: boolean): Promise<string> {
  if (shouldFail) {
    throw new Error("Something went wrong");
  }
  return "success";
}

async function main3(): Promise<void> {
  try {
    const result = await riskyOperation(true);
    console.log(result);
  } catch (err) {
    // err is `unknown` here -- you cannot call err.message directly
    // without narrowing first.
    // console.log(err.message);
    // error TS18046: 'err' is of type 'unknown'.

    if (err instanceof Error) {
      console.log(`Caught: ${err.message}`);   // Caught: Something went wrong
    } else {
      console.log("Caught a non-Error value:", err);
    }
  }
}

main3();
```

Treat every `catch` as "something was thrown, I don't yet know its
shape" — `instanceof Error` is the standard narrowing check before
touching `.message` or `.stack`.

## Modeling typed success/failure without exceptions

Throwing works, but for expected failure cases (a lookup that might not
find anything, a validation that might fail) many codebases prefer to
return a typed result instead of throwing — it forces every caller to
handle the failure case explicitly, since the type system won't let them
ignore it silently:

```typescript
type FetchResult<T> =
  | { ok: true; data: T }
  | { ok: false; error: string };

async function fetchConfigValue(key: string): Promise<FetchResult<string>> {
  const store: Record<string, string> = { theme: "dark" };
  if (key in store) {
    return { ok: true, data: store[key] };
  }
  return { ok: false, error: `No config value for "${key}"` };
}

async function main4(): Promise<void> {
  const result = await fetchConfigValue("theme");

  if (result.ok) {
    console.log(`theme = ${result.data}`);   // theme = dark
  } else {
    console.log(result.error);
  }
}

main4();
```

This is the same `Result<T, E>` shape introduced with generics in
[Module 2](02-generics.md) — async code and typed error handling combine
naturally once you have the pattern.

## A trap: forgetting `await` doesn't error, it just misbehaves

Calling an `async` function without `await` is valid TypeScript — it
returns a `Promise<T>`, which is a real value — but it's almost always a
bug:

```typescript
async function getCount(): Promise<number> {
  await delay(10);
  return 5;
}

async function main5(): Promise<void> {
  const count = getCount();   // BUG: missing `await` -- count is Promise<number>
  // console.log(count + 1);
  // error TS2365: Operator '+' cannot be applied to types 'Promise<number>' and 'number'.
  // TypeScript DOES catch this specific case because `+` needs a number --
  // but a mistake like `console.log(count)` alone would compile fine and
  // just print "Promise { <pending> }" instead of the number you expected.

  const actualCount = await getCount();
  console.log(actualCount + 1);   // 6
}

main5();
```

The safety net here is partial: TypeScript catches misuse of an unwrapped
promise *when you try to use it as the wrong type*, but a bare
`console.log(promise)` or storing it without ever awaiting it compiles
without complaint. Enable the `no-floating-promises` rule from
`typescript-eslint` in real projects — it flags promises created but
never awaited or handled.

## A trap: `async` doesn't make code run in parallel

`await`ing promises one at a time in sequence is a common performance bug
that TypeScript's types won't warn you about, since both versions type-check
identically:

```typescript
async function sequential(): Promise<number[]> {
  const a = await delay(10).then(() => 1);
  const b = await delay(10).then(() => 2);   // waits for `a` to finish first
  return [a, b];   // takes ~20ms total
}

async function parallel(): Promise<number[]> {
  const [a, b] = await Promise.all([
    delay(10).then(() => 1),
    delay(10).then(() => 2),
  ]);   // both start immediately, run concurrently
  return [a, b];   // takes ~10ms total
}
```

If the two operations don't depend on each other's results, start them
together with `Promise.all` rather than `await`ing each one in turn.

## Cheat sheet

| Concept | Syntax | Notes |
|---|---|---|
| Async function return type | `async function f(): Promise<T>` | Write the resolved type `T`, not `Promise<T>` twice |
| Unwrap inside async | `const x = await promise` | `x` has the resolved type, not `Promise<...>` |
| Run in parallel | `await Promise.all([p1, p2])` | Preserves each promise's individual type in a tuple |
| Catch type | `catch (err) { ... }` | `err` is `unknown` (TS 4.4+) — narrow with `instanceof Error` |
| Typed failure without throwing | `Promise<{ ok: true; data: T } \| { ok: false; error: string }>` | Forces callers to check `ok` before reading `data` |
| Common bug | forgetting `await` | Value stays a `Promise<T>`; TS only catches it if misused as `T` |

## Exercise

Write a function `fetchWithRetry<T>(fn: () => Promise<T>, retries:
number): Promise<T>` that calls `fn`, and if it throws, retries up to
`retries` more times before finally re-throwing the last error. Test it
against a function that fails the first two times (using a counter
closed over in the calling code) and succeeds the third time, and confirm
it resolves. Then write a `catch` block around a call that exhausts all
retries and confirm the error is narrowed with `instanceof Error` before
you print its message.
