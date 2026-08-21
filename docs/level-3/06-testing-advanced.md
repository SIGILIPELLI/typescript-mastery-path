# 06 · Testing Advanced

Level 2 covered basic Jest assertions. This module goes further: typed
mocks and spies, mocking an interface (not a concrete class), and
testing async rejection paths — using **Vitest**, which has first-class
TypeScript support and a Jest-compatible API, so everything here reads
directly onto Jest too.

## Setup

```bash
npm install -D vitest
```

## The code under test

```typescript
// notifier.ts
export interface EmailClient {
  send(to: string, subject: string, body: string): Promise<boolean>;
}

export class SignupNotifier {
  constructor(private client: EmailClient) {}

  async welcome(email: string): Promise<string> {
    const ok = await this.client.send(email, "Welcome!", "Thanks for signing up.");
    if (!ok) {
      throw new Error(`failed to email ${email}`);
    }
    return `welcomed ${email}`;
  }
}
```

`SignupNotifier` depends on an `EmailClient` **interface**, not a
concrete `SendGridClient` class — that's what makes it testable without
a real mail server: any object shaped like `EmailClient` satisfies the
constructor.

## Typed mocks, spies, and rejection testing

```typescript
// notifier.test.ts
import { describe, it, expect, vi } from "vitest";
import { SignupNotifier, EmailClient } from "./notifier";

describe("SignupNotifier", () => {
  it("returns a confirmation when the email sends", async () => {
    const client: EmailClient = {
      send: vi.fn().mockResolvedValue(true),
    };
    const notifier = new SignupNotifier(client);

    const result = await notifier.welcome("ada@example.com");

    expect(result).toBe("welcomed ada@example.com");
    expect(client.send).toHaveBeenCalledWith(
      "ada@example.com",
      "Welcome!",
      "Thanks for signing up."
    );
    expect(client.send).toHaveBeenCalledTimes(1);
  });

  it("throws when the email client reports failure", async () => {
    const client: EmailClient = {
      send: vi.fn().mockResolvedValue(false),
    };
    const notifier = new SignupNotifier(client);

    await expect(notifier.welcome("bad@example.com")).rejects.toThrow(
      "failed to email bad@example.com"
    );
  });

  it("spies without replacing behavior using a partial mock", async () => {
    const realClient: EmailClient = {
      send: async () => true,
    };
    const spy = vi.spyOn(realClient, "send");
    const notifier = new SignupNotifier(realClient);

    await notifier.welcome("grace@example.com");

    expect(spy).toHaveBeenCalledOnce();
  });
});
```

Run with `vitest run`:

```text
 RUN  v4.1.11

 Test Files  1 passed (1)
      Tests  3 passed (3)
   Start at  21:50:00
   Duration  160ms
```

`client: EmailClient = { send: vi.fn().mockResolvedValue(true) }`
type-checks because `vi.fn()` returns a mock function assignable to
`(to: string, subject: string, body: string) => Promise<boolean>` —
TypeScript checks the *shape* of the mock against the interface, so a
mock with the wrong parameter count or return type fails to compile,
not just to run.

## Traps

**`vi.fn()` alone is untyped (`Mock<any, any>`) until it's assigned into
a typed slot.** Writing `const send = vi.fn(); send(1, 2, 3);` compiles
fine on its own — the type-checking above only happens because `send`
is placed into an object literal declared as `EmailClient`. A bare
`vi.fn()` stored in an untyped `const` gives you no protection.

**`mockResolvedValue` vs. `mockReturnValue` is a common typo.** For an
`async` interface method, `mockReturnValue(true)` type-checks (a mock
function's return type is inferred loosely) but the caller's `await`
gets `true` directly rather than a resolved promise wrapping it — this
usually still works by accident because `await` on a non-promise just
returns it, but `mockResolvedValue` is the version that matches the
interface's actual `Promise<boolean>` signature and won't silently mask
a real async bug (e.g. rejection handling) the way a plain sync return
value can.

**`toHaveBeenCalledWith` doesn't fail if you pass too few expected
arguments** — it only checks the arguments you specify are among the
ones present in a lenient way for extra args in some matcher variants,
so add `toHaveBeenCalledTimes` alongside it if call count also matters,
as shown above.

**Testing against a concrete class instead of an interface forces real
mocking libraries** (`vi.mock()`, module factories) which are harder to
type correctly — depending on the `EmailClient` interface rather than a
`SendGridClient` class is what let the tests above use plain object
literals with zero mocking-framework ceremony.

## Cheat sheet

| Technique | Use for |
|---|---|
| `vi.fn().mockResolvedValue(v)` | Async method that should resolve to `v` |
| `vi.fn().mockRejectedValue(err)` | Async method that should reject |
| `vi.spyOn(obj, "method")` | Wrap a real method, keep its behavior, assert calls |
| `expect(fn).toHaveBeenCalledWith(...)` | Assert exact arguments |
| `expect(promise).rejects.toThrow(msg)` | Assert an async function throws/rejects |
| Depend on an `interface`, not a class | Enables typed object-literal mocks with no mocking library |

## Exercise

Add a `retryWelcome(email: string, attempts: number)` method to
`SignupNotifier` that calls `this.client.send` up to `attempts` times,
returning on the first success and throwing after the last failure.
Write a test using `vi.fn()` with `.mockResolvedValueOnce(false)`
chained twice then `.mockResolvedValueOnce(true)` to verify it retries
exactly the right number of times before succeeding.
