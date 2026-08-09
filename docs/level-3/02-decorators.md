# 02 · Decorators

Decorators let you attach reusable behavior to a class, method, accessor,
or field by writing `@something` right above its declaration — logging,
retry logic, validation, dependency injection, and ORMs like TypeORM all
lean on this mechanism. This module uses TypeScript's modern **standard
decorators** (the TC39 Stage 3 proposal, TypeScript's default since 5.0)
— no `experimentalDecorators` flag required, and they check cleanly
under `--strict`.

!!! note "Standard decorators vs. legacy `experimentalDecorators`"
    Older TypeScript code (and some frameworks, like older NestJS/Angular
    versions) uses a different, earlier decorator implementation enabled
    by `"experimentalDecorators": true` in `tsconfig.json`. The syntax
    looks similar but the underlying types differ. If you're adding
    decorators to an existing project and these examples don't quite
    match what you see, check for that flag first.

## Method decorators

A method decorator receives the original method and a `context` object
describing it, and returns a replacement function:

```typescript
function logged<This, Args extends unknown[], Return>(
  target: (this: This, ...args: Args) => Return,
  context: ClassMethodDecoratorContext<This, (this: This, ...args: Args) => Return>
) {
  const methodName = String(context.name);
  return function (this: This, ...args: Args): Return {
    console.log(`Calling ${methodName} with`, args);
    const result = target.call(this, ...args);
    console.log(`${methodName} returned`, result);
    return result;
  };
}

class Calculator {
  @logged
  add(a: number, b: number): number {
    return a + b;
  }
}

const calc = new Calculator();
calc.add(2, 3);
```

```text
Calling add with [ 2, 3 ]
add returned 5
```

The type parameters (`This`, `Args`, `Return`) are what keep this
generic: `logged` works on any method, on any class, with any argument
list, and the decorated method keeps its original call signature from
the outside — callers of `calc.add(2, 3)` see nothing different.

## Decorator factories — decorators that take arguments

A decorator itself can't take extra arguments directly, but a function
that *returns* a decorator can. This is a **decorator factory**, and it's
the pattern behind almost every configurable decorator you'll encounter:

```typescript
function retry(times: number) {
  return function <This, Args extends unknown[], Return>(
    target: (this: This, ...args: Args) => Return,
    context: ClassMethodDecoratorContext<This, (this: This, ...args: Args) => Return>
  ) {
    const methodName = String(context.name);
    return function (this: This, ...args: Args): Return {
      let lastError: unknown;
      for (let attempt = 1; attempt <= times; attempt++) {
        try {
          return target.call(this, ...args);
        } catch (err) {
          lastError = err;
          console.log(`${methodName} attempt ${attempt} failed`);
        }
      }
      throw lastError;
    };
  };
}

class FlakyService {
  private attempts = 0;

  @retry(3)
  fetchData(): string {
    this.attempts += 1;
    if (this.attempts < 3) {
      throw new Error(`not ready (attempt ${this.attempts})`);
    }
    return "data!";
  }
}

const service = new FlakyService();
console.log(service.fetchData());
```

```text
fetchData attempt 1 failed
fetchData attempt 2 failed
data!
```

`@retry(3)` calls `retry(3)` first, which returns the real method
decorator with `times` captured in its closure — that's the whole trick
behind "a decorator with arguments."

## Class decorators

A class decorator receives the class constructor itself and can return a
new constructor to replace it entirely — commonly used to wrap
construction with extra setup:

```typescript
function sealed<T extends new (...args: any[]) => object>(
  target: T,
  context: ClassDecoratorContext<T>
): T {
  return class extends target {
    constructor(...args: any[]) {
      super(...args);
      Object.seal(this);
    }
  };
}

@sealed
class Point {
  x = 0;
  y = 0;
}

const p = new Point();
p.x = 10;
console.log(p.x);
console.log(Object.isSealed(p));   // true -- @sealed locked the instance shape
```

```text
10
true
```

`Object.seal` is a runtime guarantee (it prevents adding new properties),
separate from TypeScript's compile-time structural checks — `@sealed`
demonstrates a decorator enforcing something at runtime that the type
system alone can't.

## Auto-accessor decorators

Fields declared with the `accessor` keyword get an implicit private
backing field plus a get/set pair, which a decorator can intercept —
useful for logging, validation, or change-tracking on a property:

```typescript
function logAccess<This, Value>(
  target: ClassAccessorDecoratorTarget<This, Value>,
  context: ClassAccessorDecoratorContext<This, Value>
): ClassAccessorDecoratorResult<This, Value> {
  const fieldName = String(context.name);
  return {
    get(this: This): Value {
      const value = target.get.call(this);
      console.log(`reading ${fieldName}:`, value);
      return value;
    },
    set(this: This, value: Value): void {
      console.log(`writing ${fieldName}:`, value);
      target.set.call(this, value);
    },
  };
}

class Counter {
  @logAccess
  accessor count = 0;
}

const counter = new Counter();
counter.count = 5;
console.log(counter.count);
```

```text
writing count: 5
reading count: 5
5
```

Note `accessor count = 0`, not `count = 0` — plain fields can only be
decorated in a much more limited way (they don't support intercepting
reads/writes like this); `accessor` is what unlocks the full
`get`/`set` decorator shape.

## A trap: decorator evaluation order surprises people

Decorators run **bottom-up** when a declaration has more than one, but
the factory calls that produce them run **top-down**:

```typescript
function first() {
  console.log("first: factory evaluated");
  return function (target: unknown, context: ClassMethodDecoratorContext) {
    console.log("first: decorator applied");
  };
}

function second() {
  console.log("second: factory evaluated");
  return function (target: unknown, context: ClassMethodDecoratorContext) {
    console.log("second: decorator applied");
  };
}

class Demo {
  @first()
  @second()
  method() {}
}
// second: factory evaluated
// first: factory evaluated
// second: decorator applied
// first: decorator applied
```

If you stack decorators (logging, then validation, then caching, for
example) and the combined behavior looks backwards, this ordering — not
a bug in your decorator — is almost always why.

## Cheat sheet

| Decorator kind | Signature receives | Typical use |
|---|---|---|
| Method | `(target, context: ClassMethodDecoratorContext)` | Logging, retry, memoization |
| Class | `(target, context: ClassDecoratorContext)` | Sealing, registration, mixins |
| Accessor | `(target, context: ClassAccessorDecoratorContext)` | Validation, change tracking |
| Field | `(value, context: ClassFieldDecoratorContext)` | Default-value transforms |
| Decorator factory | function that returns any of the above | Configurable decorators, e.g. `@retry(3)` |
| Stacking order | bottom decorator's factory runs first; decorators apply bottom-up | Matters when order affects behavior |

## Exercise

Write a method decorator factory `@memoize()` that caches a method's
return value keyed by `JSON.stringify(args)`, so repeated calls with the
same arguments skip re-running the method body (log something inside the
method so you can see it only executes once per distinct argument list).
Apply it to a deliberately slow method (e.g. one with a `for` loop doing
extra work) and call it three times with two different argument sets to
prove the cache hits.
