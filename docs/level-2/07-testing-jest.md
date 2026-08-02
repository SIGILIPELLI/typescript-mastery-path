# 07 · Testing with Jest + ts-jest

Writing TypeScript doesn't automatically mean your logic is correct — the
compiler catches type mismatches, not wrong business logic. Jest is the
most widely used JavaScript/TypeScript test runner, and `ts-jest` lets it
run `.ts` files directly, type errors and all, without a separate build
step.

## Installing and configuring

```bash
npm install --save-dev typescript jest ts-jest @types/jest
npx ts-jest config:init
```

That last command generates a `jest.config.js` wired to the `ts-jest`
preset:

```javascript
// jest.config.js
module.exports = {
  preset: "ts-jest",
  testEnvironment: "node",
};
```

`ts-jest` type-checks your test files using your project's real
`tsconfig.json` — a test with a type error fails the same way a
production file would, before it ever runs.

## Your first typed test

```typescript
// src/mathUtils.ts
export function add(a: number, b: number): number {
  return a + b;
}

export function divide(a: number, b: number): number {
  if (b === 0) {
    throw new Error("Cannot divide by zero");
  }
  return a / b;
}
```

```typescript
// __tests__/mathUtils.test.ts
import { add, divide } from "../src/mathUtils";

test("adds two numbers", () => {
  expect(add(2, 3)).toBe(5);
});

test("throws on division by zero", () => {
  expect(() => divide(1, 0)).toThrow("Cannot divide by zero");
});
```

```bash
npx jest

# PASS  __tests__/mathUtils.test.ts
#   ✓ adds two numbers
#   ✓ throws on division by zero
#
# Test Suites: 1 passed, 1 total
# Tests:       2 passed, 2 total
```

Note the `expect(() => divide(1, 0)).toThrow(...)` pattern — you must
wrap the throwing call in a function. `expect(divide(1, 0)).toThrow(...)`
would throw immediately while *evaluating the argument*, before `expect`
ever runs, and Jest wouldn't catch it as a test assertion at all.

## Typed test fixtures and `describe` blocks

```typescript
// src/user.ts
export interface User {
  id: number;
  name: string;
  active: boolean;
}

export function activeUsers(users: User[]): User[] {
  return users.filter((u) => u.active);
}
```

```typescript
// __tests__/user.test.ts
import { activeUsers, User } from "../src/user";

describe("activeUsers", () => {
  const sample: User[] = [
    { id: 1, name: "Ada", active: true },
    { id: 2, name: "Bob", active: false },
    { id: 3, name: "Cy", active: true },
  ];

  it("returns only active users", () => {
    const result = activeUsers(sample);
    expect(result).toHaveLength(2);
    expect(result.map((u) => u.name)).toEqual(["Ada", "Cy"]);
  });

  it("returns an empty array when nobody is active", () => {
    const noneActive: User[] = sample.map((u) => ({ ...u, active: false }));
    expect(activeUsers(noneActive)).toEqual([]);
  });
});
```

`sample: User[]` gets full autocomplete and type checking in the test
file, exactly as it would in production code — a typo like `activ: true`
is a compile error in the test, not a silent bug that only shows up when
the assertion mysteriously fails.

## Typed test doubles with `jest.fn()`

`jest.fn()` creates a mock function. Left untyped, calls to it are
effectively `any`; annotate it to keep the mock's shape honest:

```typescript
// src/notifier.ts
export interface Notifier {
  send(to: string, message: string): Promise<boolean>;
}

export async function notifyAll(
  notifier: Notifier,
  recipients: string[],
  message: string
): Promise<number> {
  let successCount = 0;
  for (const to of recipients) {
    const sent = await notifier.send(to, message);
    if (sent) successCount += 1;
  }
  return successCount;
}
```

```typescript
// __tests__/notifier.test.ts
import { notifyAll, Notifier } from "../src/notifier";

test("counts only successful sends", async () => {
  const mockSend = jest.fn<Promise<boolean>, [string, string]>();
  mockSend
    .mockResolvedValueOnce(true)
    .mockResolvedValueOnce(false)
    .mockResolvedValueOnce(true);

  const fakeNotifier: Notifier = { send: mockSend };

  const count = await notifyAll(fakeNotifier, ["a@x.com", "b@x.com", "c@x.com"], "Hi");

  expect(count).toBe(2);
  expect(mockSend).toHaveBeenCalledTimes(3);
  expect(mockSend).toHaveBeenNthCalledWith(1, "a@x.com", "Hi");
});
```

`jest.fn<Promise<boolean>, [string, string]>()` types the mock's return
value and parameter tuple explicitly, so `mockResolvedValueOnce(true)`
only accepts a `boolean` and a typo like `mockResolvedValueOnce("yes")`
would be a compile error, not a test that silently passes for the wrong
reason.

## Testing async code and rejected promises

```typescript
// src/fetchStatus.ts
export async function fetchStatus(shouldFail: boolean): Promise<string> {
  if (shouldFail) {
    throw new Error("Network error");
  }
  return "ok";
}
```

```typescript
// __tests__/fetchStatus.test.ts
import { fetchStatus } from "../src/fetchStatus";

test("resolves with ok on success", async () => {
  await expect(fetchStatus(false)).resolves.toBe("ok");
});

test("rejects with an error on failure", async () => {
  await expect(fetchStatus(true)).rejects.toThrow("Network error");
});
```

`.resolves` and `.rejects` unwrap a promise for you inside `expect` —
always `await` the whole `expect(...)` expression, or the test can finish
and report "passed" before the assertion even runs.

## Coverage

```bash
npx jest --coverage
```

```text
----------------|---------|----------|---------|---------|
File            | % Stmts | % Branch | % Funcs | % Lines |
----------------|---------|----------|---------|---------|
All files       |   92.3  |   85.7   |  100     |  91.6   |
 mathUtils.ts   |   100   |   100    |  100     |  100    |
 user.ts        |   100   |   100    |  100     |  100    |
 notifier.ts    |   80    |   50     |  100     |  80     |
----------------|---------|----------|---------|---------|
```

Coverage percentage is a signal, not a goal to game — 100% coverage with
assertions that never check meaningful behavior (`expect(true).toBe(true)`
inside every function) is worse than useful 80% coverage. Use the report
to find code paths nobody exercises, not as a score to maximize.

## A trap: `any` in test helpers defeats the whole exercise

It's tempting to type test setup loosely to "get it working faster," but
that's exactly where bad data quietly slips through untested:

```typescript
// Weak: `any` means a wrong shape here wouldn't be caught until the
// function under test happens to touch the missing/wrong field.
function makeUser(overrides: any = {}) {
  return { id: 1, name: "Test", active: true, ...overrides };
}

// Better: overrides are constrained to a real subset of User.
function makeTypedUser(overrides: Partial<User> = {}): User {
  return { id: 1, name: "Test", active: true, ...overrides };
}

// makeTypedUser({ name: 123 });
// error TS2322: Type 'number' is not assignable to type 'string'.
// The `any` version above would accept this silently.
```

Factory functions like `makeTypedUser` are common in larger test suites —
keep them typed with `Partial<T>` (from [Module 5](05-utility-types.md))
so a typo in a test's setup fails at compile time, not three tests later
with a confusing runtime error.

## Cheat sheet

| Task | API |
|---|---|
| Define a test | `test("name", () => { ... })` or `it("name", () => { ... })` |
| Group related tests | `describe("group", () => { ... })` |
| Basic equality | `expect(x).toBe(y)` (primitives) / `expect(x).toEqual(y)` (objects/arrays) |
| Expect a throw | `expect(() => fn()).toThrow("message")` |
| Async resolve/reject | `await expect(promise).resolves.toBe(x)` / `.rejects.toThrow(...)` |
| Typed mock function | `jest.fn<ReturnType, [ArgTypes]>()` |
| Queue a mock's async result | `mockFn.mockResolvedValueOnce(value)` |
| Run with coverage | `npx jest --coverage` |

## Exercise

Write `src/cart.ts` exporting a `Cart` class with `addItem(name: string,
price: number): void`, `removeItem(name: string): void`, and `total():
number`. Write `__tests__/cart.test.ts` with a `describe("Cart")` block
covering: adding a single item updates the total correctly, adding
multiple items sums correctly, removing an item that exists reduces the
total, and removing an item that doesn't exist leaves the total
unchanged. Run `npx jest --coverage` and confirm `cart.ts` shows 100%
statement coverage.
