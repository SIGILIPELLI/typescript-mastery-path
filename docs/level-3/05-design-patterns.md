# 05 · Design Patterns in TypeScript

Classic OOP patterns read differently in TypeScript than in Java or C++
because interfaces are structural and functions are first-class — you
often don't need the ceremony the Gang of Four book uses. This module
implements five patterns you'll actually run into in real codebases:
Singleton, Factory, Strategy, Observer, and Builder.

## Singleton

```typescript
class ConfigStore {
  private static instance: ConfigStore;
  private values = new Map<string, string>();
  private constructor() {}

  static getInstance(): ConfigStore {
    if (!ConfigStore.instance) {
      ConfigStore.instance = new ConfigStore();
    }
    return ConfigStore.instance;
  }

  set(key: string, value: string): void {
    this.values.set(key, value);
  }
  get(key: string): string | undefined {
    return this.values.get(key);
  }
}

ConfigStore.getInstance().set("env", "production");
console.log(ConfigStore.getInstance().get("env"));
console.log(ConfigStore.getInstance() === ConfigStore.getInstance());
```

```text
production
true
```

The `private constructor()` is what makes this a real singleton at the
type level — `new ConfigStore()` from outside the class is a compile
error, so `getInstance()` is the only way in.

## Factory

```typescript
interface Shape {
  area(): number;
}
class Circle implements Shape {
  constructor(private radius: number) {}
  area(): number {
    return Math.PI * this.radius ** 2;
  }
}
class Square implements Shape {
  constructor(private side: number) {}
  area(): number {
    return this.side ** 2;
  }
}

type ShapeKind = "circle" | "square";
function createShape(kind: ShapeKind, size: number): Shape {
  switch (kind) {
    case "circle":
      return new Circle(size);
    case "square":
      return new Square(size);
  }
}

console.log(createShape("circle", 2).area().toFixed(2));
console.log(createShape("square", 2).area());
```

```text
12.57
4
```

Because `kind` is a union of string literals, `switch` over it is
exhaustive — add a third `ShapeKind` and TypeScript will flag
`createShape` as possibly not returning a value on every path (with
`noImplicitReturns`), catching the missed `case` at compile time.

## Strategy

```typescript
interface DiscountStrategy {
  apply(total: number): number;
}
const noDiscount: DiscountStrategy = { apply: (t) => t };
const tenPercentOff: DiscountStrategy = { apply: (t) => t * 0.9 };

class Cart {
  constructor(private strategy: DiscountStrategy) {}
  checkout(total: number): number {
    return this.strategy.apply(total);
  }
}

console.log(new Cart(noDiscount).checkout(100));
console.log(new Cart(tenPercentOff).checkout(100));
```

```text
100
90
```

No abstract base class needed — `DiscountStrategy` is structural, so
any object shaped like `{ apply(total: number): number }` qualifies,
including a plain object literal as shown here.

## Observer

```typescript
interface Observer<T> {
  notify(value: T): void;
}
class Subject<T> {
  private observers: Observer<T>[] = [];
  subscribe(o: Observer<T>): void {
    this.observers.push(o);
  }
  publish(value: T): void {
    for (const o of this.observers) o.notify(value);
  }
}

const priceFeed = new Subject<number>();
priceFeed.subscribe({ notify: (v) => console.log("logger sees", v) });
priceFeed.subscribe({ notify: (v) => console.log("alerter sees", v) });
priceFeed.publish(42);
```

```text
logger sees 42
alerter sees 42
```

`Subject<T>` being generic means the same class works for a number feed,
a string feed, or a feed of custom event objects — `Observer<T>` and
`Subject<T>` stay linked so a `Subject<number>` rejects an observer
expecting `string`.

## Builder

```typescript
class RequestBuilder {
  private method = "GET";
  private headers: Record<string, string> = {};
  private url = "";

  setMethod(method: string): this {
    this.method = method;
    return this;
  }
  setUrl(url: string): this {
    this.url = url;
    return this;
  }
  addHeader(key: string, value: string): this {
    this.headers[key] = value;
    return this;
  }
  build() {
    return { method: this.method, url: this.url, headers: this.headers };
  }
}

const req = new RequestBuilder()
  .setMethod("POST")
  .setUrl("/users")
  .addHeader("Content-Type", "application/json")
  .build();
console.log(req);
```

```text
{
  method: 'POST',
  url: '/users',
  headers: { 'Content-Type': 'application/json' }
}
```

The `this` return type (not `RequestBuilder`) matters if a subclass adds
more chained methods — with `this`, the subclass's own chain methods
keep returning the subclass type, so a subclassed builder doesn't lose
its extra methods midway through a chain.

## Traps

**A Factory `switch` without `noImplicitReturns` won't catch a missing
case** — TypeScript infers the return type includes `undefined` on the
unhandled branch only if you've turned that flag on; otherwise a typo'd
or added `ShapeKind` value silently returns `undefined` at runtime with
no compile error.

**Singleton state leaks across tests.** Because `ConfigStore.instance`
is a `static`, every test in the same process shares it — a common
source of "works alone, fails in the suite" bugs. Add a
`static resetForTests()` escape hatch if you use this pattern in code
under test.

**`this` return types don't survive being reassigned to a
`RequestBuilder`-typed variable.** If you write
`const b: RequestBuilder = new RequestBuilder(); b.setMethod("POST")`,
the chain still works, but any subclass-specific method called after
that point on `b` is invisible — the variable's declared type wins.

## Cheat sheet

| Pattern | TypeScript feature that makes it clean |
|---|---|
| Singleton | `private constructor` + `private static instance` |
| Factory | Discriminated string-literal union + exhaustive `switch` |
| Strategy | Structural interfaces — no base class required |
| Observer | Generic `Subject<T>` / `Observer<T>` pair |
| Builder | `this`-typed chained methods |

## Exercise

Implement the **Decorator pattern** (the GoF structural pattern, not
TypeScript's `@decorator` syntax) for the `Shape` interface: write a
`BorderedShape` class that wraps another `Shape`, implements `Shape`
itself, and returns `wrapped.area() + borderPadding`. Compose two
levels deep (`new BorderedShape(new BorderedShape(new Circle(2), 1), 1)`)
and log the result to confirm both paddings apply.
