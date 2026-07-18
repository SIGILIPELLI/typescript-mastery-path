# 06 · Arrays & Objects, Typed

TypeScript lets you describe exactly what's inside an array or object, so the
compiler catches mismatched-shape bugs before you ever run the code.

## Typed arrays

```typescript
const scores: number[] = [90, 85, 77];
const names: Array<string> = ["Ada", "Grace"];   // equivalent generic syntax

scores.push(92);     // fine -- number
// scores.push("x"); // error TS2345: Argument of type 'string' is not assignable to parameter of type 'number'

console.log(scores[0]);        // 90
console.log(scores.length);    // 4
```

`number[]` and `Array<number>` mean exactly the same thing — `T[]` is just
shorthand for `Array<T>`.

## Tuples — fixed-length arrays with per-position types

```typescript
let point: [number, number] = [3, 4];
let entry: [string, number] = ["Ada", 30];

console.log(point[0]);   // 3, typed as number
console.log(entry[0]);   // "Ada", typed as string
console.log(entry[1]);   // 30, typed as number

// entry = [30, "Ada"];  // error TS2322: types don't match the declared order
```

A tuple looks like an array but tracks a specific type at each index —
useful for fixed-shape pairs/triples like coordinates or `[key, value]`
entries.

## Typed objects (inline shape)

```typescript
const person: { name: string; age: number } = {
  name: "Ada",
  age: 30,
};

console.log(person.name);   // Ada
// person.email = "x";       // error TS2339: Property 'email' does not exist
```

Inline object types work, but for anything reused more than once, prefer a
named `interface` or `type` (covered in Module 5) instead of repeating the
inline shape everywhere.

## Array methods keep their types

```typescript
const numbers: number[] = [1, 2, 3, 4, 5];

const doubled: number[] = numbers.map((n) => n * 2);
const evens: number[] = numbers.filter((n) => n % 2 === 0);
const total: number = numbers.reduce((sum, n) => sum + n, 0);

console.log(doubled);   // [2, 4, 6, 8, 10]
console.log(evens);     // [2, 4]
console.log(total);     // 15
```

`.map`, `.filter`, and `.reduce` infer their return types from the callback
you pass — TypeScript figures out `doubled` is `number[]` without you writing
it explicitly.

## Object destructuring with types

```typescript
interface Config {
  host: string;
  port: number;
  debug?: boolean;
}

function connect({ host, port, debug = false }: Config): void {
  console.log(`Connecting to ${host}:${port}, debug=${debug}`);
}

connect({ host: "localhost", port: 8080 });
// Connecting to localhost:8080, debug=false
```

## Index signatures — objects with dynamic keys

```typescript
const prices: { [productName: string]: number } = {
  apple: 1.5,
  banana: 0.75,
};

prices.cherry = 3.0;   // adding a new key is fine -- shape matches the index signature
console.log(prices["apple"]);   // 1.5
```

An index signature (`{ [key: string]: number }`) is how you type an object
you're using as a dictionary/map, where you don't know every key up front.

## Cheat sheet

| Feature | Syntax |
|---------|--------|
| Typed array | `number[]` or `Array<number>` |
| Tuple | `[string, number]` |
| Inline object type | `{ name: string; age: number }` |
| Optional property | `age?: number` |
| Index signature | `{ [key: string]: number }` |
| Destructure with default | `function f({ x = 0 }: { x?: number }) {}` |

## Exercise

Define a tuple type `Coordinate = [number, number]`. Write a function
`distance(a: Coordinate, b: Coordinate): number` computing the Euclidean
distance between two points. Then write a function
`summarizePrices(prices: { [item: string]: number }): { total: number;
average: number }` that returns the total and average of all values in the
dictionary.
