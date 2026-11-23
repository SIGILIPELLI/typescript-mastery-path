---
title: "TypeScript Cheat Sheet: Quick Reference, Beginner to Advanced"
description: "A free TypeScript cheat sheet: 35 quick-reference tables from every level of the TypeScript Mastery Path, each linked to its full lesson."
---

# TypeScript Cheat Sheet

Quick-reference tables from **35 lessons** of the free TypeScript Mastery Path, in course order from beginner to advanced. Each section links to the full lesson with explanations and runnable examples.

## Level 1 · Entry

### [02 · Basic Types](level-1/02-basic-types.md)

| Type | Example | Notes |
|------|---------|-------|
| `string` | `"hello"` | Text |
| `number` | `42`, `3.14` | All numbers -- no int/float split |
| `boolean` | `true`, `false` | |
| `T[]` / `Array<T>` | `number[]` | Homogeneous array |
| `[T, U]` | `[string, number]` | Fixed-length tuple |
| `any` | -- | Disables checking -- avoid |
| `unknown` | -- | Safe "don't know yet" -- requires narrowing |
| `null` / `undefined` | -- | Only assignable elsewhere via a union under `strict` |
| `void` | -- | Function returns nothing meaningful |

### [03 · Control Flow with Types](level-1/03-control-flow.md)

| Construct | Notes |
|-----------|-------|
| `if`/`else` | Narrows union types per-branch via control flow analysis |
| `switch` | Same narrowing; combine with `never` for exhaustiveness checks |
| Truthiness check | Excludes `0`, `""`, `NaN`, `null`, `undefined` -- be explicit if 0/"" are valid |
| `for...of` | Iterate values (arrays, strings, most iterables) |
| `for...in` | Iterate keys (objects) |
| `never` | Type of a value that can't occur -- powers exhaustiveness checks |

### [04 · Functions &amp; Type Annotations](level-1/04-functions-type-annotations.md)

| Feature | Syntax | Notes |
|---------|--------|-------|
| Parameter type | `(x: number)` | Required unless annotated optional |
| Return type | `): number {` | TypeScript infers if omitted, but annotate public functions |
| Optional param | `(title?: string)` | Must follow required params; type includes `undefined` |
| Default param | `(exp: number = 2)` | Optional + supplies a fallback value |
| Rest param | `(...nums: number[])` | Collects trailing args into an array |
| Function type | `(a: number, b: number) => number` | Type of a callback / function value |
| Overloads | multiple signatures + one implementation | For calls that differ in argument shape |

### [05 · Interfaces &amp; Type Aliases](level-1/05-interfaces-type-aliases.md)

| Feature | Syntax | Notes |
|---------|--------|-------|
| Interface | `interface X { ... }` | Best for object shapes, can be extended/merged |
| Type alias | `type X = { ... }` | Can name unions, tuples, primitives too |
| Optional prop | `age?: number` | May be omitted; type includes `undefined` |
| Readonly prop | `readonly id: string` | Settable once, then immutable |
| `extends` | `interface B extends A` | Interface inheritance |
| Structural typing | -- | Shape match is enough, no explicit `implements` needed |

### [06 · Arrays &amp; Objects, Typed](level-1/06-arrays-objects-typed.md)

| Feature | Syntax |
|---------|--------|
| Typed array | `number[]` or `Array<number>` |
| Tuple | `[string, number]` |
| Inline object type | `{ name: string; age: number }` |
| Optional property | `age?: number` |
| Index signature | `{ [key: string]: number }` |
| Destructure with default | `function f({ x = 0 }: { x?: number }) {}` |

### [07 · Classes Basics](level-1/07-classes-basics.md)

| Feature | Syntax |
|---------|--------|
| Class | `class Name { ... }` |
| Constructor shorthand | `constructor(public x: T) {}` |
| Access modifiers | `public` / `private` / `protected` / `readonly` |
| Inheritance | `class Sub extends Base { ... }` |
| Call parent constructor | `super(args)` |
| Getter/setter | `get prop() { ... }` / `set prop(v) { ... }` |

### [08 · Enums](level-1/08-enums.md)

| Kind | When to use |
|------|-------------|
| Numeric enum | Rarely needed now; reverse-mapping is the main reason to pick it |
| String enum | Readable, serializable — a solid default when you want an enum |
| `const enum` | Same as string/numeric, but compiled away for smaller output |
| String literal union | Zero runtime cost, often simpler for small fixed sets |

### [09 · Modules &amp; tsconfig Deep Dive](level-1/09-modules-tsconfig.md)

| Task | Syntax/Command |
|------|-----------------|
| Named export | `export function name() {}` |
| Named import | `import { name } from "./file";` |
| Default export | `export default function name() {}` |
| Default import | `import anyName from "./file";` |
| Re-export everything | `export * from "./file";` |
| Generate tsconfig | `npx tsc --init` |
| Compile | `npx tsc` |
| Run without a separate compile step | `npx ts-node file.ts` |

## Level 2 · Intermediate

### [01 · Union &amp; Intersection Types](level-2/01-union-intersection-types.md)

| Concept | Syntax | Meaning |
|---|---|---|
| Union | `A \| B` | Value is one of these types |
| Intersection | `A & B` | Value satisfies all of these types |
| Discriminant | shared literal field, e.g. `kind: "circle"` | Lets TS narrow a union automatically |
| Exhaustiveness check | `const x: never = value` in a `default` | Compiler error if a case is missing |
| Primitive intersection | `string & number` | Resolves to `never` -- no value can satisfy both |
| Excess property check | only on fresh object literals | Assign via a variable to bypass it |

### [02 · Generics](level-2/02-generics.md)

| Syntax | Meaning |
|---|---|
| `function f<T>(x: T): T` | Generic function, `T` inferred from the call site |
| `class Box<T> { ... }` | Generic class, instantiated as `new Box<number>(...)` |
| `T extends HasLength` | Constrain `T` to types with a `length` property |
| `K extends keyof T` | Constrain `K` to an actual key of `T` |
| `Box<T = unknown>` | Default type parameter if none is given |
| `T[]` / `Array<T>` | Equivalent generic array syntaxes |

### [03 · Interfaces vs Type Aliases Deep Dive](level-2/03-interfaces-vs-types.md)

| Feature | `interface` | `type` |
|---|---|---|
| Object shapes | Yes | Yes |
| Unions / tuples / primitives | No | Yes |
| Declaration merging | Yes (same name merges) | No (duplicate name errors) |
| Combine with conflict detection | `extends` (errors early) | `&` (resolves to `never` silently) |
| Generic | Yes | Yes |
| `implements` in a class | Yes | Yes |

### [04 · Classes Advanced](level-2/04-classes-advanced.md)

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

### [05 · Utility Types](level-2/05-utility-types.md)

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

### [06 · Async/Await with Types](level-2/06-async-await-types.md)

| Concept | Syntax | Notes |
|---|---|---|
| Async function return type | `async function f(): Promise<T>` | Write the resolved type `T`, not `Promise<T>` twice |
| Unwrap inside async | `const x = await promise` | `x` has the resolved type, not `Promise<...>` |
| Run in parallel | `await Promise.all([p1, p2])` | Preserves each promise's individual type in a tuple |
| Catch type | `catch (err) { ... }` | `err` is `unknown` (TS 4.4+) — narrow with `instanceof Error` |
| Typed failure without throwing | `Promise<{ ok: true; data: T } \| { ok: false; error: string }>` | Forces callers to check `ok` before reading `data` |
| Common bug | forgetting `await` | Value stays a `Promise<T>`; TS only catches it if misused as `T` |

### [07 · Testing with Jest + ts-jest](level-2/07-testing-jest.md)

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

### [08 · Working with JSON/APIs](level-2/08-working-with-json-apis.md)

| Step | Code | Why |
|---|---|---|
| Raw fetch result | `await response.json()` | Type is `any` — trusts nothing |
| Assertion (no check) | `(await response.json()) as T` | Compiles, but performs zero runtime verification |
| Status check | `if (!response.ok) throw ...` | A 404/500 still resolves; only network failure rejects |
| Safe starting type | `const data: unknown = ...` | Forces a check before use, unlike `any` |
| Real validation | `function isT(x: unknown): x is T` | A type guard that actually inspects the shape |
| Reusable pattern | `fetchJson<T>(url, isT)` | Generic helper combining fetch + status check + validation |
| Array validation | `Array.isArray(x) && x.every(isT)` | Checks both "is an array" and "every element matches" |

### [09 · Type Narrowing &amp; Type Guards](level-2/09-type-narrowing-guards.md)

| Guard | Example | Best for |
|---|---|---|
| `typeof` | `typeof x === "string"` | Primitives |
| `instanceof` | `x instanceof MyClass` | Class instances, check subclasses before base classes |
| `in` | `"prop" in x` | Distinguishing object shapes without a discriminant |
| Discriminated union | `switch (x.kind) { ... }` | Unions with a shared literal tag field |
| `Array.isArray` | `Array.isArray(x)` | Distinguishing an array from other objects |
| User-defined guard | `function isX(v): v is X { ... }` | Anything the built-ins can't express directly |
| `value is T` return type | required for a custom guard | Without it, TS treats the function as a plain `boolean` and narrows nothing |

## Level 3 · Advanced

### [01 · Advanced Generics](level-3/01-advanced-generics.md)

| Feature | Syntax | Use it for |
|---|---|---|
| Conditional type | `T extends U ? X : Y` | Branch on a type at compile time |
| Infer | `T extends (infer U)[] ? U : T` | Pull a type out of a wrapper type |
| Distribution | `T extends unknown ? T[] : never` | Applies the conditional per union member |
| Block distribution | `[T] extends [unknown] ? T[] : never` | Treats a union as one type |
| Mapped type | `{ [K in keyof T]: T[K] }` | Transform every property of `T` |
| Remove a modifier | `{ -readonly [K in keyof T]-?: T[K] }` | Strip `readonly`/`?` off a mapped type |
| Key remapping | `[K in keyof T as \`get${string & K}\`]` | Rename properties while mapping |
| Template literal type | `` `on${Capitalize<Event>}` `` | Build a string-literal union from parts |
| Variadic tuple | `[...T, ...U]` | Concatenate/prepend tuple types exactly |

### [02 · Decorators](level-3/02-decorators.md)

| Decorator kind | Signature receives | Typical use |
|---|---|---|
| Method | `(target, context: ClassMethodDecoratorContext)` | Logging, retry, memoization |
| Class | `(target, context: ClassDecoratorContext)` | Sealing, registration, mixins |
| Accessor | `(target, context: ClassAccessorDecoratorContext)` | Validation, change tracking |
| Field | `(value, context: ClassFieldDecoratorContext)` | Default-value transforms |
| Decorator factory | function that returns any of the above | Configurable decorators, e.g. `@retry(3)` |
| Stacking order | bottom decorator's factory runs first; decorators apply bottom-up | Matters when order affects behavior |

### [03 · Building APIs with Express + TS](level-3/03-building-apis-express-ts.md)

| Type | Purpose |
|---|---|
| `Request<P>` | Types `req.params` as `P` |
| `Request<P, ResBody, ReqBody>` | Also types `req.body` |
| `Response<ResBody>` | Constrains what `res.json()` accepts |
| `NextFunction` | Type of Express's `next` callback |
| `(err, req, res, next) => void` (4 args) | Signature Express recognizes as an error handler |
| `express.json()` | Middleware that populates `req.body` from JSON |

### [04 · Type-Safe Database Access](level-3/04-type-safe-database-access.md)

| Pattern | What it buys you |
|---|---|
| `class Table<T extends Row>` | One repository implementation, reused per entity |
| `where<K extends keyof T>(key: K, value: T[K])` | Column name and value type stay linked |
| `Omit<T, "id">` for inserts | Callers can't (and don't need to) supply an id |
| `Partial<Omit<T, "id">>` for updates | Any subset of non-id fields, nothing else |
| `sql<T>` tagged template | Attaches a result type to a query without changing its runtime shape |

### [05 · Design Patterns in TypeScript](level-3/05-design-patterns.md)

| Pattern | TypeScript feature that makes it clean |
|---|---|
| Singleton | `private constructor` + `private static instance` |
| Factory | Discriminated string-literal union + exhaustive `switch` |
| Strategy | Structural interfaces — no base class required |
| Observer | Generic `Subject<T>` / `Observer<T>` pair |
| Builder | `this`-typed chained methods |

### [06 · Testing Advanced](level-3/06-testing-advanced.md)

| Technique | Use for |
|---|---|
| `vi.fn().mockResolvedValue(v)` | Async method that should resolve to `v` |
| `vi.fn().mockRejectedValue(err)` | Async method that should reject |
| `vi.spyOn(obj, "method")` | Wrap a real method, keep its behavior, assert calls |
| `expect(fn).toHaveBeenCalledWith(...)` | Assert exact arguments |
| `expect(promise).rejects.toThrow(msg)` | Assert an async function throws/rejects |
| Depend on an `interface`, not a class | Enables typed object-literal mocks with no mocking library |

### [07 · Build Tooling](level-3/07-build-tooling.md)

| Tool | Job | Type-checks? | Bundles? |
|---|---|---|---|
| `tsc` | Compile + type-check | Yes | No |
| `tsc --noEmit` | Type-check only (CI gate) | Yes | No |
| esbuild | Fast bundle/transpile | No | Yes |
| tsup / Vite | esbuild-powered build tool with sane defaults | No (delegates to `tsc` separately) | Yes |
| `"moduleResolution": "bundler"` | Match bundler-style import resolution | — | — |

### [08 · Third-Party Types](level-3/08-third-party-types.md)

| Situation | Fix |
|---|---|
| Untyped npm package | `npm install -D @types/<package>` if it exists |
| Untyped local `.js` file | Add a sibling `.d.ts` file with matching name |
| Untyped package, no `@types` exists | Add `declare module "package-name" { ... }` in a project-level `.d.ts` |
| Adding a method to a built-in inside a module file | `declare global { interface X { ... } }` |
| Extending a third-party exported interface | A second `interface` declaration with the same name (merges) |

### [09 · Performance &amp; Compilation](level-3/09-performance-compilation.md)

| Setting/flag | Effect |
|---|---|
| `--extendedDiagnostics` | Prints parse/bind/check/emit timing breakdown |
| `skipLibCheck: true` | Skips re-checking `.d.ts` files; usually the biggest win |
| `incremental: true` | Caches check results in `.tsbuildinfo` between CLI runs |
| Project references + `tsc --build` | Per-package incremental checking in a monorepo |
| `TS2589` | Recursion depth limit hit on a conditional/mapped type |

## Level 4 · Master

### [01 · Advanced Type-Level Programming](level-4/01-advanced-type-level-programming.md)

| Feature | What it does |
|---|---|
| `` `${A}${B}` `` (template literal type) | Concatenates string-literal types |
| `Capitalize<T>` / `Uppercase` / `Lowercase` / `Uncapitalize` | Built-in string-literal transforms |
| `S extends \`${infer Head}${D}${infer Rest}\`` | Captures parts of a matched string type |
| `T extends unknown[] ? T["length"] : never` | Reads a tuple's length as a numeric literal type |
| `function assertNever(x: never): never` | Compile-time exhaustiveness check |

### [02 · Framework Integration Patterns](level-4/02-framework-integration-patterns.md)

| Pattern | Where it shows up |
|---|---|
| `class App<State>` with a `Context<State>` | Koa-style typed app state |
| `type Middleware<State> = (ctx, next) => void` | Any onion-style middleware chain |
| `interface BasePlugin<State>` matching the app's generic | Plugins that stay in sync with app state |
| Guard against double `next()` calls | Prevents silent double-processing bugs |
| Declaration merging (Level 3, Module 08) | Extending an existing framework's types, not building your own |

### [03 · Monorepo &amp; Large-Scale Structure](level-4/03-monorepo-large-scale-structure.md)

| Concern | Solved by |
|---|---|
| "Which package depends on which" for `tsc --build` | `references` in each `tsconfig.json` |
| Compiler resolving `@acme/core` while editing/type-checking | `paths` |
| Node resolving `@acme/core` at runtime | Workspace linking (npm/pnpm/yarn workspaces) |
| Skipping unchanged packages on rebuild | `composite: true` + `.tsbuildinfo` per package |
| A package other packages can reference | `"composite": true`, `"declaration": true` |

### [04 · Testing at Scale &amp; CI](level-4/04-testing-at-scale-ci.md)

| Tool/config | Purpose |
|---|---|
| `@vitest/coverage-v8` | Adds `--coverage` reporting to Vitest |
| `coverage.thresholds` in `vitest.config.ts` | Fails the run below a coverage floor |
| Separate `typecheck` and `test` npm scripts | Vitest doesn't type-check; `tsc --noEmit` must run separately |
| `strategy.matrix.node-version` in Actions | Runs the full suite across multiple Node versions |
| `npm ci` (not `npm install`) in CI | Reproducible installs; fails on lockfile drift |

### [05 · Security Considerations](level-4/05-security-considerations.md)

| Pattern | Prevents |
|---|---|
| `type SafeHtml = Brand<string, "SafeHtml">` | Passing unsanitized strings where sanitized ones are required |
| `type PositiveInt = Brand<number, "PositiveInt">` | Passing unvalidated numbers into functions with numeric invariants |
| Validating `loadEnv()` instead of casting `process.env.X as string` | `undefined` config silently reaching production code |
| Re-validating branded values at trust boundaries | Assuming a brand survived serialization |
| `=== undefined` instead of falsy checks for optional strings | Rejecting valid-but-falsy values like `"0"` |

### [06 · Deployment with Docker](level-4/06-deployment-docker.md)

| Dockerfile technique | Why it matters for TypeScript specifically |
|---|---|
| Multi-stage build | Compiler and dev deps stay out of the shipped image |
| `npm ci --omit=dev` in the final stage | Excludes `typescript`, `@types/*` from production |
| `COPY --from=build /app/dist ./dist` | Ships compiled JS, never `.ts` source |
| `package*.json` copied before source | Preserves the `npm ci` layer cache across source-only changes |
| `.dockerignore` with `node_modules`, `dist` | Prevents stale local artifacts leaking into the build context |
| `USER node` | Runs the container process unprivileged |

### [07 · Performance Optimization](level-4/07-performance-optimization.md)

| Technique | Solves | Typed via |
|---|---|---|
| `memoize<Args, Result>` | Repeated calls with the same expensive arguments | Generic over parameter tuple and return type |
| Generators (`function*`) | Avoiding full-array allocation for large/infinite sequences | `Generator<T>` / `Iterable<T>` |
| `Pool<T>` | GC churn from many short-lived objects in hot loops | Generic factory + typed acquire/release |
| `--target es2020`+ | Enables direct `for...of` over generic `Iterable<T>` | Compiler flag, not a code change |

### [08 · Writing Robust Public APIs](level-4/08-writing-robust-public-apis.md)

| Pattern | Use for |
|---|---|
| `Result<T, E>` | Public functions where callers must handle failure explicitly |
| Function overloads | Different argument counts/types that mean genuinely different things |
| `Readonly<T>` return type | Preventing accidental mutation of shared/default objects |
| Explicit return-type annotations on helpers | Keeping discriminated unions narrow and correct |
| `DeepReadonly<T>` (custom) | Full immutability for nested return values |

### [09 · Strict Mode Best Practices](level-4/09-strict-mode-best-practices.md)

| Flag | In `strict`? | Catches |
|---|---|---|
| `noImplicitAny` | Yes | Untyped parameters/variables defaulting to `any` |
| `strictNullChecks` | Yes | Using a possibly-`null`/`undefined` value without a check |
| `strictPropertyInitialization` | Yes | Class fields never assigned in the constructor |
| `strictFunctionTypes` | Yes | Unsound function parameter variance |
| `noUncheckedIndexedAccess` | **No** | Index/array access assumed always present |
| `noImplicitOverride` | **No** | Subclass method silently no longer overriding anything |
| `exactOptionalPropertyTypes` | **No** | Blurring "key absent" vs. "key present, value `undefined`" |

---

Go deeper with the full [TypeScript Mastery Path](index.md) curriculum or test yourself with the [interview questions](interview-questions.md).
