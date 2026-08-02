# 04 · Classes Advanced

[Level 1](../level-1/07-classes-basics.md) covered typed properties,
constructors, and basic `public`/`private`/`protected` modifiers. This
module builds on that: abstract classes that can't be instantiated
directly, `implements` versus `extends`, static members, and a few access
modifier subtleties that trip people up in real projects.

## Abstract classes

An `abstract class` defines a partial implementation that subclasses must
complete. You can't create an instance of it directly:

```typescript
abstract class Shape {
  abstract area(): number;   // no body -- subclasses must implement it

  describe(): string {
    return `This shape has an area of ${this.area().toFixed(2)}`;
  }
}

class Circle extends Shape {
  constructor(private radius: number) {
    super();
  }

  area(): number {
    return Math.PI * this.radius ** 2;
  }
}

const c = new Circle(3);
console.log(c.describe());   // This shape has an area of 28.27

// new Shape();
// error TS2511: Cannot create an instance of an abstract class.
```

`describe()` is a **concrete method** shared by all subclasses; `area()`
is an **abstract method** — a contract with no implementation, forcing
every subclass to supply its own. This is the key difference from a plain
interface: an abstract class can mix shared, working code with
must-override contracts in the same declaration.

## `implements` vs `extends`

```typescript
interface Flyable {
  fly(): string;
}

abstract class Bird {
  constructor(protected species: string) {}
  abstract sound(): string;
}

class Sparrow extends Bird implements Flyable {
  constructor() {
    super("Sparrow");
  }

  sound(): string {
    return "Chirp!";
  }

  fly(): string {
    return `${this.species} flies away`;
  }
}

const sparrow = new Sparrow();
console.log(sparrow.sound());   // Chirp!
console.log(sparrow.fly());     // Sparrow flies away
```

A class can `extends` **at most one** class (single inheritance) but can
`implements` **any number** of interfaces — use `extends` for "is a kind
of, and inherits behavior," and `implements` for "conforms to this
contract" without inheriting any code.

## Static members

`static` properties and methods belong to the class itself, not to
instances:

```typescript
class Counter {
  private static count = 0;

  constructor() {
    Counter.count += 1;
  }

  static get total(): number {
    return Counter.count;
  }
}

new Counter();
new Counter();
new Counter();

console.log(Counter.total);   // 3
// new Counter().count would be an error -- count is only on the class
```

Static members are useful for shared state (a registry, a counter) or
factory methods that build instances without exposing every constructor
detail:

```typescript
class Point {
  private constructor(public x: number, public y: number) {}

  static origin(): Point {
    return new Point(0, 0);
  }

  static fromArray([x, y]: [number, number]): Point {
    return new Point(x, y);
  }
}

const p1 = Point.origin();
const p2 = Point.fromArray([3, 4]);
console.log(p1, p2);   // Point { x: 0, y: 0 } Point { x: 3, y: 4 }

// new Point(1, 1);
// error TS2673: Constructor of class 'Point' is private and only
// accessible within the class declaration.
```

A `private constructor()` forces every instance to go through a named
static factory — a common pattern when you want to validate or normalize
inputs before an object can exist at all.

## `readonly` vs `private` — different jobs

They're often used together but solve different problems:

```typescript
class Config {
  readonly version: string;      // can be READ from outside, never reassigned
  private secretKey: string;     // cannot be accessed from outside at all

  constructor(version: string, secretKey: string) {
    this.version = version;
    this.secretKey = secretKey;
  }

  maskedKey(): string {
    return this.secretKey.slice(0, 2) + "***";
  }
}

const cfg = new Config("1.0.0", "sk_live_abc123");
console.log(cfg.version);      // 1.0.0 -- readable from outside
console.log(cfg.maskedKey());  // sk***

// cfg.version = "2.0.0";
// error TS2540: Cannot assign to 'version' because it is a read-only property.
// console.log(cfg.secretKey);
// error TS2341: Property 'secretKey' is private and only accessible within class 'Config'.
```

`readonly` controls **mutability**; `private`/`protected` control
**visibility**. A property can be both `private readonly` — set once in
the constructor and never touched again, by anyone, including the class
itself after construction.

## A trap: access modifiers are compile-time only

TypeScript's `private` and `protected` are erased when compiled to
JavaScript — they exist purely for the type checker, not as runtime
enforcement:

```typescript
class Wallet {
  private balance = 100;
}

const w = new Wallet();
// w.balance;
// error TS2341 at compile time...

// ...but the compiled JavaScript has no such protection:
console.log((w as any).balance);   // 100 -- the "private" field is fully
                                    // readable once you cast past the
                                    // type checker
```

If you need runtime-enforced privacy (not just compile-time), use a real
JavaScript private field with the `#` prefix instead — those *are*
enforced by the JS engine itself, not just by TypeScript:

```typescript
class TrueWallet {
  #balance = 100;

  deposit(amount: number): void {
    this.#balance += amount;
  }

  get balance(): number {
    return this.#balance;
  }
}

const tw = new TrueWallet();
console.log(tw.balance);   // 100
// (tw as any).#balance would still be a syntax error -- #-fields are
// only accessible from inside the class body, enforced by JavaScript
// itself, no cast can reach them.
```

## Overriding methods and calling `super`

```typescript
class Employee {
  constructor(protected name: string, protected baseSalary: number) {}

  pay(): number {
    return this.baseSalary;
  }
}

class Manager extends Employee {
  constructor(name: string, baseSalary: number, private bonus: number) {
    super(name, baseSalary);
  }

  override pay(): number {
    return super.pay() + this.bonus;   // reuse the parent's logic, then add to it
  }
}

const mgr = new Manager("Dee", 60000, 15000);
console.log(mgr.pay());   // 75000
```

The `override` keyword (available since TypeScript 4.3) is optional but
recommended: it tells the compiler "I intend to override a base class
method," and if the base class method's name or signature ever changes or
is removed, you get a compile error instead of a silently unused method.

## Cheat sheet

| Feature | Syntax | Notes |
|---|---|---|
| Abstract class | `abstract class X { abstract m(): T; }` | Cannot be instantiated; subclasses must implement abstract members |
| Interface implementation | `class X implements I` | Any number of interfaces; contract only, no inherited code |
| Single inheritance | `class X extends Y` | At most one base class |
| Static member | `static prop` / `static method()` | Belongs to the class, not instances |
| Private constructor | `private constructor() {}` | Forces construction through static factories |
| Compile-time privacy | `private` / `protected` | Erased at runtime; a cast to `any` reaches them |
| Runtime privacy | `#field` | Enforced by JavaScript itself, even past a cast |
| Explicit override | `override method()` | Compiler checks the base method still exists |

## Exercise

Design an `abstract class PaymentMethod` with an abstract method
`charge(amount: number): string` and a concrete method `receipt(amount:
number): string` that calls `charge` and wraps the result. Create two
subclasses, `CreditCard` (constructor takes a masked card number) and
`PayPal` (constructor takes an email), each implementing `charge`
differently. Add a `static` counter on `PaymentMethod` that tracks how
many payment method instances have been created across all subclasses,
and print the total after creating a few of each.
