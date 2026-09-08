# 07 · Classes Basics

TypeScript classes look like JavaScript classes with typed properties, typed
constructor parameters, and access modifiers the compiler actually enforces.

## Defining a class with typed properties

```typescript
class Person {
  name: string;
  age: number;

  constructor(name: string, age: number) {
    this.name = name;
    this.age = age;
  }

  greet(): string {
    return `Hi, I'm ${this.name} and I'm ${this.age} years old.`;
  }
}

const ada = new Person("Ada", 30);
console.log(ada.greet());   // Hi, I'm Ada and I'm 30 years old.
```

## Parameter properties — a shorthand constructor

```typescript
class Person {
  constructor(
    public name: string,
    public age: number
  ) {}

  greet(): string {
    return `Hi, I'm ${this.name}`;
  }
}

const grace = new Person("Grace", 40);
console.log(grace.greet());   // Hi, I'm Grace
```

Prefixing a constructor parameter with `public`, `private`, or `protected`
declares AND assigns the property in one step — no separate `this.name =
name` line needed.

## Access modifiers

```typescript
class BankAccount {
  private balance: number;

  constructor(public owner: string, initialBalance: number) {
    this.balance = initialBalance;
  }

  deposit(amount: number): void {
    this.balance += amount;
  }

  getBalance(): number {
    return this.balance;
  }
}

const account = new BankAccount("Ada", 100);
account.deposit(50);
console.log(account.getBalance());   // 150
// console.log(account.balance);      // error TS2341: Property 'balance' is private
```

| Modifier | Visible from |
|----------|--------------|
| `public` (default) | Anywhere |
| `private` | Only inside this class |
| `protected` | This class and subclasses |
| `readonly` | Anywhere, but can't be reassigned after construction |

## Inheritance with `extends`

```typescript
class Animal {
  constructor(public name: string) {}

  makeSound(): string {
    return "...";
  }
}

class Dog extends Animal {
  makeSound(): string {
    return "Woof!";
  }
}

class Cat extends Animal {
  makeSound(): string {
    return "Meow!";
  }
}

const animals: Animal[] = [new Dog("Rex"), new Cat("Whiskers")];

for (const animal of animals) {
  console.log(`${animal.name}: ${animal.makeSound()}`);
}
// Rex: Woof!
// Whiskers: Meow!
```

## Calling the parent constructor with `super`

```typescript
class Animal {
  constructor(public name: string) {}
}

class Dog extends Animal {
  constructor(name: string, public breed: string) {
    super(name);   // must call super() before using `this` in a subclass
  }
}

const rex = new Dog("Rex", "Labrador");
console.log(`${rex.name} is a ${rex.breed}`);   // Rex is a Labrador
```

## Getters and setters

```typescript
class Temperature {
  private _celsius: number = 0;

  get celsius(): number {
    return this._celsius;
  }

  set celsius(value: number) {
    if (value < -273.15) {
      throw new Error("Below absolute zero");
    }
    this._celsius = value;
  }

  get fahrenheit(): number {
    return this._celsius * 9 / 5 + 32;
  }
}

const temp = new Temperature();
temp.celsius = 25;
console.log(temp.fahrenheit);   // 77
```

`get`/`set` let you use property-like syntax (`temp.celsius = 25`) while
still running validation logic behind the scenes.

## How It Actually Works

TypeScript classes compile to plain JS `class` syntax (or, on older `target`s, to constructor-function-plus-prototype patterns), with every type annotation stripped: `private`, `public`, `protected`, parameter-property shorthand, and any `: Type` annotation on a field all vanish from the emitted code. This means `private` in TypeScript is a **compile-time-only** access restriction — the checker refuses code outside the class that reads `instance.secret`, but the emitted JS field is an ordinary, fully-accessible property, so `instance["secret"]` (bracket access with a string) or plain JS calling code bypasses the restriction entirely at runtime. True runtime privacy requires the JS-native `#field` syntax, which TypeScript also supports and which *does* enforce access at the engine level because it's a distinct kind of property key, not compiler-checked.

Despite being structurally typed everywhere else, TypeScript classes are compared **structurally, with one nominal-like exception**: two classes with identical public members are mutually assignable even if unrelated by inheritance — but a `private` or `protected` member makes the comparison behave nominally for that pair, because the checker additionally requires the member to originate from the *same declaration* (the same class or a shared ancestor), not just a matching name and type. This is why two classes that both declare a `private id: string` field are *not* assignable to each other even though the public shape looks identical — the checker treats each class's private members as tagged with that specific class's identity.

Parameter properties (`constructor(private name: string) {}`) are pure syntactic sugar: the compiler expands that single parameter into a declared class field plus an assignment statement (`this.name = name;`) inserted at the top of the constructor body — nothing new is introduced with respect to `private`'s erasure, it's just fewer characters to type; the underlying execution is identical to writing the field and the assignment out by hand.

## Cheat sheet

| Feature | Syntax |
|---------|--------|
| Class | `class Name { ... }` |
| Constructor shorthand | `constructor(public x: T) {}` |
| Access modifiers | `public` / `private` / `protected` / `readonly` |
| Inheritance | `class Sub extends Base { ... }` |
| Call parent constructor | `super(args)` |
| Getter/setter | `get prop() { ... }` / `set prop(v) { ... }` |

## Exercise

Write a class `Shape` with a `protected` method `area(): number` that returns
`0` by default. Create subclasses `Circle` (constructor takes a `radius`) and
`Rectangle` (constructor takes `width` and `height`), each overriding `area()`
correctly. Put instances of both in a `Shape[]` array and print each one's
area using a loop.
