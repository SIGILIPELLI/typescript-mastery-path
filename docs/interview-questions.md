---
title: "TypeScript Interview Questions and Answers (179 Questions)"
description: "179 TypeScript interview questions with answers and explanations, from beginner to advanced, organised by level. Free, from the TypeScript Mastery Path."
---

# TypeScript Interview Questions and Answers

**179 questions** with the correct answer and a short explanation, organised from beginner (Level 1) to advanced (Level 4). Every question comes from a lesson in the free TypeScript Mastery Path, so you can open the lesson to go deeper.

!!! tip "Practise, don't just read"
    Each lesson also ends with an interactive quiz, and the [final quiz](quiz.md) tests all four levels. Try to answer each question below before reading the answer.

## Level 1 · Entry

### [01 · Setup &amp; First Program](level-1/01-setup.md)

**Q1.** What does tsc do to your TypeScript code?

**Answer:** Type-checks it and erases all types, emitting plain JavaScript

Types exist only at compile time and leave no trace in the output.

**Q2.** Why run the compiler with npx tsc for a project-local install?

**Answer:** npx finds the project-local binary instead of assuming it's on the global PATH

A per-project install is the standard for real projects.

**Q3.** What is the point of the error TS2345 when you call greet(42) on a function taking a string?

**Answer:** It catches the bug before the program runs

tsc refuses to compile until the error is fixed unless told otherwise.

**Q4.** What does npx tsc --init create?

**Answer:** A tsconfig.json describing the project's files, target and strictness

Real projects use tsconfig.json rather than passing a filename.

### [02 · Basic Types](level-1/02-basic-types.md)

**Q5.** How many numeric types does TypeScript have for ordinary numbers?

**Answer:** One: number (plus bigint for arbitrarily large integers)

There is no separate int/float/double.

**Q6.** What does `let city = "Seattle";` infer, and what happens on `city = 42`?

**Answer:** Infers string; assigning 42 is error TS2322

Inference works from the initializer without an annotation.

**Q7.** What is a tuple?

**Answer:** An array with a known length and a known type at each position

Example: [string, number, boolean]. Labels like [status: number, body: string] are documentation only.

**Q8.** What does annotating a value as any do?

**Answer:** Disables type checking for that value, so even calling a number compiles

Mistakes on any values are not caught at compile time.

### [03 · Control Flow with Types](level-1/03-control-flow.md)

**Q9.** Inside `if (typeof id === "number")` for id: string | number, what is id's type?

**Answer:** number, because TypeScript narrows it

In the else branch it narrows to string.

**Q10.** What is this ability to track types through branches called?

**Answer:** Control flow analysis

It's why type-safe code needs fewer manual casts.

**Q11.** What does formatId(42) return in the lesson?

**Answer:** "42"

id.toFixed(0) on 42 gives the string "42"; formatId("ab12") gives "AB12".

**Q12.** What does a switch on a union of string literals such as Direction let the compiler do?

**Answer:** Check which cases are handled, since each case narrows the value

Control flow tracks the remaining possible values.

### [04 · Functions &amp; Type Annotations](level-1/04-functions-type-annotations.md)

**Q13.** What does power(5) return for `function power(base: number, exponent: number = 2)`?

**Answer:** 25

The default exponent of 2 applies; power(5, 3) is 125.

**Q14.** Where must optional parameters appear?

**Answer:** After all required parameters

TypeScript enforces this at the declaration site.

**Q15.** In `function sum(...numbers: number[])`, what does the number[] annotation describe?

**Answer:** The collected array of trailing arguments

sum(10, 20, 30, 40) returns 100.

**Q16.** Why annotate return types on functions other code depends on?

**Answer:** It documents intent and catches drift between implementation and callers' expectations

TypeScript usually infers it, but explicit annotation is good practice.

### [05 · Interfaces &amp; Type Aliases](level-1/05-interfaces-type-aliases.md)

**Q17.** What happens with `const bad: User = { id: 2, name: "Bob" }` when User requires email?

**Answer:** Error TS2741: Property 'email' is missing

An object literal must have the declared properties.

**Q18.** What does a readonly property like sku prevent?

**Answer:** Reassignment after creation (error TS2540)

It can be set once.

**Q19.** What does item.discount evaluate to when an optional discount is omitted?

**Answer:** undefined

Optional properties default to undefined when absent.

**Q20.** What is the common rule of thumb for interface versus type?

**Answer:** interface for extendable object shapes and public APIs; type for unions, tuples and non-object shapes

Level 2 covers declaration merging and extends versus &amp;.

### [06 · Arrays &amp; Objects, Typed](level-1/06-arrays-objects-typed.md)

**Q21.** How do number[] and Array&lt;number&gt; relate?

**Answer:** They mean exactly the same thing

T[] is shorthand for Array&lt;T&gt;.

**Q22.** What does `const evens = numbers.filter((n) => n % 2 === 0)` produce for [1,2,3,4,5]?

**Answer:** [2, 4]

The callback keeps even numbers; and reduce sums to 15.

**Q23.** What does connect({ host: "localhost", port: 8080 }) print for a destructured debug = false default?

**Answer:** Connecting to localhost:8080, debug=false

The default applies when debug is omitted.

**Q24.** Why prefer a named interface over repeating an inline object type?

**Answer:** It's reusable and stays consistent when the shape changes

Inline shapes work but get repeated.

### [07 · Classes Basics](level-1/07-classes-basics.md)

**Q25.** What does `constructor(public name: string, public age: number) {}` do?

**Answer:** Declares and assigns the properties in one step

No separate this.name = name line is needed.

**Q26.** What happens with `account.balance` when balance is private?

**Answer:** Error TS2341: Property 'balance' is private

The class exposes getBalance() instead.

**Q27.** Which modifier makes a member visible in the class and its subclasses?

**Answer:** protected

public is the default; readonly blocks reassignment after construction.

**Q28.** What does the loop over [new Dog("Rex"), new Cat("Whiskers")] print?

**Answer:** Rex: Woof! then Whiskers: Meow!

Each subclass overrides makeSound().

### [08 · Enums](level-1/08-enums.md)

**Q29.** With `enum Direction { North, South, East, West }`, what is Direction.South and Direction[1]?

**Answer:** 1 and "South"

Numeric enums start at 0 and are reverse-mappable.

**Q30.** Why are string enums usually the better default?

**Answer:** They serialise to readable values like "ACTIVE", which is easier to debug and safer to store or send

They have no numeric reverse mapping.

**Q31.** What does handle(HttpStatus.NotFound) print with NotFound = 404?

**Answer:** Error: 404

The status isn't HttpStatus.OK, so the else branch runs.

**Q32.** What is the trade-off of a const enum?

**Answer:** It's inlined at compile time with no runtime object, so you can't iterate members or reverse lookup

The output is smaller.

### [09 · Modules &amp; tsconfig Deep Dive](level-1/09-modules-tsconfig.md)

**Q33.** Why are named exports generally preferred over default exports?

**Answer:** They're easier to refactor and better supported by auto-import tooling

A default import can be given any name without the compiler noticing a different meaning.

**Q34.** What does strict: true enable?

**Answer:** All strict type-checking flags, including strictNullChecks and noImplicitAny

The lesson calls it the single most important setting.

**Q35.** What do outDir and rootDir configure?

**Answer:** Where compiled .js is written and where the compiler expects .ts source

In the lesson, rootDir is ./src and outDir is ./dist.

**Q36.** What does skipLibCheck do?

**Answer:** Skips type-checking .d.ts files from dependencies, for faster builds

It's one of the key compilerOptions.

### [10 · Project — Typed CLI To-Do App](level-1/10-project-todo-cli.md)

**Q37.** How does the app persist tasks between runs?

**Answer:** It reads and writes a JSON file (tasks.json) with fs

loadTasks returns [] if the file is missing or can't be parsed.

**Q38.** How does the app number tasks for the user?

**Answer:** Displayed 1-based (i + 1) and converted with index - 1 internally

completeTask reads tasks[index - 1].

**Q39.** What does deleteTask do for an out-of-range number?

**Answer:** Prints "No task with number N" and returns

It checks index &lt; 1 || index &gt; tasks.length.

**Q40.** What does `JSON.parse(contents) as Task[]` do for type safety?

**Answer:** Asserts the type at compile time only; it doesn't validate the data at runtime

Types are erased, so a malformed file is not detected by the assertion.

## Level 2 · Intermediate

### [01 · Union &amp; Intersection Types](level-2/01-union-intersection-types.md)

**Q41.** On a value of type string | number, which members can you use without narrowing?

**Answer:** Only those common to every type in the union

value.length alone errors with TS2339 because number has no length.

**Q42.** What is a discriminated union?

**Answer:** A union of object types sharing a literal property (like kind) that lets TypeScript narrow by checking it

The shared kind literal is the discriminant.

**Q43.** What does area({ kind: "square", side: 3 }) return in the lesson?

**Answer:** 9

The non-circle branch returns shape.side ** 2; the circle with radius 2 gives 12.57.

**Q44.** What does printLength(3.14159) print?

**Answer:** 3

It narrows to number and prints value.toFixed(0) = "3".

### [02 · Generics](level-2/02-generics.md)

**Q45.** Why is firstAny(arr: any[]): any risky?

**Answer:** Calling oops.toUpperCase() on a number compiles but crashes at runtime

any disables checks, so the bug hides until runtime.

**Q46.** What does `firstElement([1, 2, 3])` infer as the type of num?

**Answer:** number

T is inferred from the argument; you rarely need firstElement&lt;number&gt;(...).

**Q47.** What does pair("age", 30) have as its type?

**Answer:** [string, number], a tuple

Two type parameters A and B keep each position's type.

**Q48.** What does `T extends HasLength` achieve?

**Answer:** It constrains T to types that have a numeric length, so logLength(42) is an error

Strings, arrays and { length: 10 } all pass.

### [03 · Interfaces vs Type Aliases Deep Dive](level-2/03-interfaces-vs-types.md)

**Q49.** What happens with two interfaces of the same name in the same scope?

**Answer:** They merge automatically

Two type aliases of the same name give TS2300 Duplicate identifier.

**Q50.** What practical use does declaration merging have?

**Answer:** Extending third-party types such as Express's Request without editing the library

It's also a trap if you accidentally name an interface like one from a library.

**Q51.** For a plain object shape, are UserI (interface) and UserT (type) assignable to each other?

**Answer:** Yes: structural typing doesn't care which form produced the shape

printUser(asType) works in the lesson.

**Q52.** How do interfaces and type aliases combine with others?

**Answer:** Interfaces use extends; type aliases use &amp;, and they behave differently on conflicting members

A conflicting extends gives TS2430.

### [04 · Classes Advanced](level-2/04-classes-advanced.md)

**Q53.** What happens with `new Shape()` when Shape is abstract?

**Answer:** Error TS2511: Cannot create an instance of an abstract class

Subclasses must implement abstract methods like area().

**Q54.** How does an abstract class differ from a plain interface?

**Answer:** It can mix shared working code with must-override contracts

describe() is concrete while area() is abstract.

**Q55.** How many classes can a class extend, and how many interfaces can it implement?

**Answer:** At most one class; any number of interfaces

extends is for 'is a kind of'; implements for 'conforms to this contract'.

**Q56.** What does Counter.total print after three `new Counter()` calls?

**Answer:** 3

The static count belongs to the class, not instances.

### [05 · Utility Types](level-2/05-utility-types.md)

**Q57.** What does Partial&lt;Article&gt; make each property?

**Answer:** Optional, so callers can send only the changed fields

Useful for update functions.

**Q58.** What happens with run({ timeout: 1000 }) when run takes Required&lt;Options&gt;?

**Answer:** Error: Property 'retries' is missing

Required forces every optional property to be present.

**Q59.** Does Readonly&lt;Article&gt; stop draft.tags.push(...)?

**Answer:** No: it's compile-time only and shallow, so tags is read-only as a reference but not deeply frozen

It doesn't deep-freeze the object at runtime.

**Q60.** What does Pick&lt;Article, "id" | "title" | "tags"&gt; produce?

**Answer:** A type with only those three properties

Accessing preview.body is error TS2339.

### [06 · Async/Await with Types](level-2/06-async-await-types.md)

**Q61.** What type do you write for `async function getGreeting()` that returns a plain string?

**Answer:** Promise&lt;string&gt;

TypeScript wraps a plain return in a Promise automatically.

**Q62.** What does `await` do to a Promise&lt;T&gt; inside an async function?

**Answer:** Unwraps it to T

Outside async functions you use .then() on the Promise&lt;T&gt;.

**Q63.** How does Promise.all type the result of [Promise&lt;string&gt;, Promise&lt;number&gt;, Promise&lt;boolean&gt;]?

**Answer:** As a tuple [string, number, boolean]

Each element keeps its own type.

**Q64.** What is the type of the caught value in a catch block since TypeScript 4.4 (by default)?

**Answer:** unknown, forcing you to narrow

It was any previously; unknown is a deliberate safety improvement.

### [07 · Testing with Jest + ts-jest](level-2/07-testing-jest.md)

**Q65.** What does ts-jest add to Jest?

**Answer:** It runs .ts files directly and type-checks tests with your real tsconfig.json

A test with a type error fails like a production file would.

**Q66.** Why must you write expect(() =&gt; divide(1, 0)).toThrow(...) rather than expect(divide(1, 0)).toThrow(...)?

**Answer:** The second throws while evaluating the argument, before expect runs

Wrap the throwing call in a function.

**Q67.** Given activeUsers(sample) with Ada (active), Bob (inactive), Cy (active), what does result.map(u =&gt; u.name) equal?

**Answer:** ["Ada", "Cy"]

The filter keeps active users, so the length is 2.

**Q68.** What does the compiler catch versus Jest tests?

**Answer:** The compiler catches type mismatches; tests catch wrong business logic

TypeScript doesn't make logic correct.

### [08 · Working with JSON/APIs](level-2/08-working-with-json-apis.md)

**Q69.** What type does response.json() give, and what does TypeScript check?

**Answer:** any; nothing is verified against the real response

returning data as Todo trusts you completely.

**Q70.** What does `as Todo` do at runtime?

**Answer:** Nothing: it performs zero runtime checking

It's better than any but is not validation.

**Q71.** When does fetch reject its promise?

**Answer:** Only on a network failure; 404 and 500 still resolve

Check response.ok, which is false outside 200–299.

**Q72.** How do you really verify API data is a Todo?

**Answer:** Validate the shape at runtime with a type guard

The lesson's Step 3 uses a hand-written guard.

### [09 · Type Narrowing &amp; Type Guards](level-2/09-type-narrowing-guards.md)

**Q73.** What does describeValue(3.14159) print?

**Answer:** number: 3.14

The typeof number branch uses toFixed(2).

**Q74.** What is narrowing?

**Answer:** The compiler tracking that a variable's type became more specific, from the control flow you wrote

No extra annotations are needed.

**Q75.** What does formatValue(undefined) return for string | string[] | undefined?

**Answer:** (none)

After excluding undefined and string, the last option is string[] and uses join.

**Q76.** Why can a top-level function named describe fail with @types/jest installed?

**Answer:** It collides with Jest's global describe as a duplicate identifier

Watch for names that overlap with globals your type setup provides.

### [10 · Project — Typed Weather Dashboard](level-2/10-project-weather-dashboard.md)

**Q77.** Why does the project use Open-Meteo?

**Answer:** It's free and needs no API key or signup

It has a geocoding endpoint (city to coordinates) and a forecast endpoint.

**Q78.** How are the Jest tests kept free of network calls?

**Answer:** They cover only the pure data-shaping logic, such as summary and weatherCodes

summary.ts is a pure function combining location and forecast.

**Q79.** What happens to API responses before they're trusted?

**Answer:** Both are validated at runtime

The project applies the Level 2 JSON-validation lesson.

**Q80.** Which pieces does the capstone combine?

**Answer:** Generics, utility types, typed async/await, JSON validation and Jest tests

It also caches repeated city lookups.

## Level 3 · Advanced

### [01 · Advanced Generics](level-3/01-advanced-generics.md)

**Q81.** What is IsString&lt;number&gt; in `type IsString<T> = T extends string ? "yes" : "no"`?

**Answer:** "no"

It's resolved entirely by the compiler; nothing remains at runtime.

**Q82.** What does ElementType&lt;number[]&gt; give with `T extends (infer U)[] ? U : T`?

**Answer:** number

infer pulls U out of the array type; ElementType&lt;string&gt; falls through to string.

**Q83.** What does Unwrap&lt;Promise&lt;number&gt;&gt; give?

**Answer:** number

It's the same pattern as the built-in Awaited&lt;T&gt;.

**Q84.** What trap does a conditional type with a naked type parameter have when given a union?

**Answer:** It distributes over each member separately and unions the results

ToArray&lt;string | number&gt; becomes string[] | number[].

### [02 · Decorators](level-3/02-decorators.md)

**Q85.** Which decorators does the module use?

**Answer:** Standard decorators (TC39 Stage 3, default since TypeScript 5.0), with no experimentalDecorators flag

Older NestJS/Angular code may use the legacy flag.

**Q86.** What does a method decorator receive and return?

**Answer:** The original method and a context object; it returns a replacement function

The generic This, Args and Return keep the method's signature.

**Q87.** What does calc.add(2, 3) print with @logged?

**Answer:** Calling add with [ 2, 3 ] then add returned 5

The wrapper logs before and after calling the target.

**Q88.** What is a decorator factory?

**Answer:** A function that returns a decorator, which lets decorators take arguments, like retry(times)

It's the pattern behind almost every configurable decorator.

### [03 · Building APIs with Express + TS](level-3/03-building-apis-express-ts.md)

**Q89.** What happens to req and res without @types/express?

**Answer:** They fall back to any, so TypeScript can't help

The package gives requests, responses and middleware a real shape.

**Q90.** What does Request&lt;{}, {}, CreateUserBody&gt; type?

**Answer:** req.body, since the third type parameter is the body

Request&lt;{ id: string }&gt; types req.params; the middle parameter is the response body.

**Q91.** What status does POST /users return on success, and on a missing name or email?

**Answer:** 201 on success; 400 when name or email is missing

A missing user on GET /users/:id returns 404, and the error handler returns 500.

**Q92.** What identifies Express's error-handling middleware?

**Answer:** It takes four parameters: err, req, res, next

It's registered last with app.use(errorHandler).

### [04 · Type-Safe Database Access](level-3/04-type-safe-database-access.md)

**Q93.** Why is `users.where("active", "yes")` a compile error?

**Answer:** where&lt;K extends keyof T&gt;(key: K, value: T[K]) requires the value to match that property's type (boolean)

This is the trick Prisma and Drizzle use to type-check where() calls.

**Q94.** What does users.findById(99) return in the lesson?

**Answer:** undefined

Its return type is T | undefined.

**Q95.** What does insert(data: Omit&lt;T, "id"&gt;) mean?

**Answer:** The caller supplies everything except id, which the table assigns

A nextId counter assigns it.

**Q96.** Why build a Table&lt;T&gt; by hand?

**Answer:** So the types in ORMs like Prisma, Drizzle and Knex stop feeling like magic

It's an in-memory mirror of what those libraries generate.

### [05 · Design Patterns in TypeScript](level-3/05-design-patterns.md)

**Q97.** What makes ConfigStore a singleton at the type level?

**Answer:** A private constructor, so getInstance() is the only way in

new ConfigStore() from outside is a compile error.

**Q98.** What does ConfigStore.getInstance() === ConfigStore.getInstance() print?

**Answer:** true

Both calls return the same stored instance.

**Q99.** Which five patterns does the module cover?

**Answer:** Singleton, Factory, Strategy, Observer, Builder

They're the ones met in real codebases.

**Q100.** Why does the ceremony of the Gang of Four book often shrink in TypeScript?

**Answer:** Interfaces are structural and functions are first-class

Patterns read differently than in Java or C++.

### [06 · Testing Advanced](level-3/06-testing-advanced.md)

**Q101.** Why does SignupNotifier depend on an EmailClient interface rather than a concrete class?

**Answer:** Any object shaped like EmailClient satisfies it, so tests need no real mail server

That's what makes it testable.

**Q102.** What does `vi.fn().mockResolvedValue(false)` make the fake send return?

**Answer:** A promise resolving to false, so welcome() throws "failed to email …"

The test asserts with await expect(...).rejects.toThrow(...).

**Q103.** What does vi.spyOn(realClient, "send") do?

**Answer:** Records calls while keeping the real behaviour

The test then expects the spy toHaveBeenCalledOnce().

**Q104.** Why does the lesson use Vitest?

**Answer:** It has first-class TypeScript support and a Jest-compatible API, so it reads onto Jest too

Install with npm install -D vitest.

### [07 · Build Tooling](level-3/07-build-tooling.md)

**Q105.** What does esbuild not do?

**Answer:** Type-check; it will bundle code with type errors

It strips types and transpiles without verifying them.

**Q106.** What does tsc not do?

**Answer:** Bundle: each input file becomes its own output file

That's fine for Node but not for shipping a single script to a browser.

**Q107.** What does the standard combination use each tool for?

**Answer:** tsc --noEmit as a type-check gate; esbuild for fast bundling

The build:safe script runs typecheck then build.

**Q108.** Why is esbuild so much faster than tsc in the lesson (8ms vs about 240ms)?

**Answer:** It's written in Go and skips type-checking

The gap grows on big projects.

### [08 · Third-Party Types](level-3/08-third-party-types.md)

**Q109.** What happens when you import a JS module that has no types?

**Answer:** Every export gets type any, until you add a .d.ts

legacy/index.d.ts matches legacy/index.js by file name.

**Q110.** What are @types/* packages?

**Answer:** Community-maintained types from DefinitelyTyped, picked up automatically from node_modules/@types

For example @types/lodash alongside lodash.

**Q111.** Why is `declare global { … }` needed to add shout() to String inside a file with an import or export?

**Answer:** The file is a module, so a plain interface String would only shadow locally

declare global is the escape hatch back to global scope.

**Q112.** What is declaration merging?

**Answer:** Two same-named interfaces combine into one interface

It's the mechanism behind module augmentation.

### [09 · Performance &amp; Compilation](level-3/09-performance-compilation.md)

**Q113.** Why does tsc report 83 files and 58,456 lines for one small .ts file?

**Answer:** lib.d.ts and other built-in definitions are pulled in and type-checked by default

Check time, not your own line count, dominates.

**Q114.** What did skipLibCheck do to Check time in the lesson's measurement?

**Answer:** Dropped it from 0.132s to 0.001s

Types fell from 35,374 to 395, and your own code is still fully checked.

**Q115.** What does incremental: true do?

**Answer:** Writes a .tsbuildinfo file so later runs re-check only changed files and their dependents

Configured with tsBuildInfoFile.

**Q116.** Which flag measures where compile time goes?

**Answer:** --extendedDiagnostics

It prints files, lines, types, instantiations, memory and check time.

### [10 · Project — REST API with Express](level-3/10-project-rest-api-express.md)

**Q117.** How do NotFoundError and ValidationError help the API?

**Answer:** Custom Error subclasses are thrown by the store and mapped to status codes by a central error handler

find() throws NotFoundError if a task is missing.

**Q118.** What does TaskStore.update(id, patch) do with an UpdateTaskBody?

**Answer:** Changes only the fields provided (title and/or done)

Each field is checked against undefined.

**Q119.** How does the build and run flow work?

**Answer:** tsc compiles src to dist; node dist/index.js starts the server

dev uses tsc --watch.

**Q120.** What does list() return, and why?

**Answer:** A copy ([...this.tasks]) so callers can't mutate the store's array

It protects the private state.

## Level 4 · Master

### [01 · Advanced Type-Level Programming](level-4/01-advanced-type-level-programming.md)

**Q121.** What is EventName&lt;"click"&gt; for `on${Capitalize<T>}`?

**Answer:** "onClick"

Capitalize is one of four intrinsic string types, alongside Uppercase, Lowercase and Uncapitalize.

**Q122.** What does Split&lt;"a.b.c", "."&gt; give?

**Answer:** ["a", "b", "c"]

It recurses on Rest until the pattern no longer matches, then the base case [S] wraps what's left.

**Q123.** What does `infer` capture inside a template literal pattern?

**Answer:** A piece of the matched string as its own type variable

It's head-recursion at the type level.

**Q124.** What does the dot-path type enable?

**Answer:** Autocompleting and type-checking keys like "server.port" against a real object shape

It's built from template literal types, recursive conditional types and tuples.

### [02 · Framework Integration Patterns](level-4/02-framework-integration-patterns.md)

**Q125.** What shape do Express, Koa and Fastify share?

**Answer:** A chain of middleware functions passed a context and a next callback

The module builds that pattern from scratch.

**Q126.** Why is App&lt;State&gt; generic?

**Answer:** Each project picks its own State, and every middleware's ctx.state is then typed with no casts

This is how ctx.state (Koa) or req.user (Express via augmentation) can be typed per project.

**Q127.** What do ctx.state.logs and ctx.state.userId hold after app.handle(ctx) runs on "/dashboard"?

**Answer:** ['request to /dashboard', 'handled for user-42'] and user-42

The three middlewares run in order, each calling next().

**Q128.** What does the run function's check `if (i <= index)` guard against?

**Answer:** next() being called multiple times

It throws "next() called multiple times".

### [03 · Monorepo &amp; Large-Scale Structure](level-4/03-monorepo-large-scale-structure.md)

**Q129.** What does composite: true require and enable?

**Answer:** It's required on every package a reference points at, forcing declaration: true and stricter incremental bookkeeping

It lets other packages depend on this one's compiled .d.ts output.

**Q130.** What does paths in tsconfig do?

**Answer:** Tells the type checker where to resolve @acme/core; it doesn't affect what Node does at runtime

Runtime resolution is handled by the package manager's workspace linking.

**Q131.** What do references do?

**Answer:** Describe the dependency graph for tsc --build: build core before api and rebuild api if core changes

They solve type-checking; workspace linking solves resolution.

**Q132.** What makes the root tsconfig a pure solution file?

**Answer:** files: [] plus references, so tsc --build builds every package in dependency order

It has no code of its own.

### [04 · Testing at Scale &amp; CI](level-4/04-testing-at-scale-ci.md)

**Q133.** Which test covers the `if (!ok) throw` branch of notifier.ts?

**Answer:** The failure-path test (rejects.toThrow); without it Branches would read 50% (1/2)

Every statement, branch, function and line was covered at 100%.

**Q134.** What does vitest run --coverage use in the lesson?

**Answer:** @vitest/coverage-v8

It's configured with provider: "v8" and reporters like text and lcov.

**Q135.** Why enforce a coverage floor in config?

**Answer:** So the run fails if coverage drops below the thresholds

It's wired into the CI workflow.

**Q136.** What does the CI workflow do in this module?

**Answer:** Runs the tests with coverage on every push on GitHub Actions, across multiple Node versions

Hundreds of tests in CI are a different problem from a few local tests.

### [05 · Security Considerations](level-4/05-security-considerations.md)

**Q137.** What does a branded type like SafeHtml do?

**Answer:** Makes a plain string not assignable, so unescaped input passed to renderToPage is a compile error

The brand exists purely in the type system and costs nothing at runtime.

**Q138.** How can a SafeHtml value be produced?

**Answer:** Only by calling escapeHtml (which asserts the brand)

As long as escapeHtml is correct, every call site is forced through it.

**Q139.** What does renderToPage(escapeHtml("&lt;script&gt;alert('xss')&lt;/script&gt;")) output?

**Answer:** &lt;div&gt;&amp;lt;script&amp;gt;alert('xss')&amp;lt;/script&amp;gt;&lt;/div&gt;

The angle brackets are escaped.

**Q140.** What else does the branded-type pattern generalise to?

**Answer:** ValidatedEmail, HashedPassword, TrustedUrl and PositiveInt

Anywhere raw and checked values of the same primitive could be mixed up.

### [06 · Deployment with Docker](level-4/06-deployment-docker.md)

**Q141.** What is wrong with the naive Dockerfile (COPY . . then npm install)?

**Answer:** The image ships dev dependencies, the compiler and full source, adding size and attack surface

These provide zero runtime value.

**Q142.** What does the multi-stage build's production stage contain?

**Answer:** Only production dependencies (npm ci --omit=dev) and the compiled dist copied with COPY --from=build

No compiler, no .ts source, no dev tooling.

**Q143.** Why have a .dockerignore even though the Dockerfile copies specific paths?

**Answer:** Docker still uploads the whole directory to the daemon first, including stale node_modules or dist

It also keeps .env and .git out of the context.

**Q144.** What do ENV NODE_ENV=production and USER node accomplish?

**Answer:** Production mode and running as a non-root user

EXPOSE 4500 documents the port.

### [07 · Performance Optimization](level-4/07-performance-optimization.md)

**Q145.** Why does memoize&lt;Args extends unknown[], Result&gt; keep the wrapped function's exact signature?

**Answer:** It's generic over both the parameter tuple and the return type

fastSquare is still (n: number) =&gt; number.

**Q146.** What is the default cache key in the lesson's memoize?

**Answer:** JSON.stringify(args)

The keyFn parameter can override it.

**Q147.** What does take(lazyRange(0, 1_000_000), 5) print, and why is it cheap?

**Answer:** [ 0, 1, 2, 3, 4 ]; the generator produces values on demand without a million-element array

take stops pulling after 5 values.

**Q148.** What is the key point of typing performance helpers well?

**Answer:** TypeScript compiles away, so the speed-ups are JavaScript techniques, but good generic types keep the wrapped signature

Otherwise the helper becomes a liability.

### [08 · Writing Robust Public APIs](level-4/08-writing-robust-public-apis.md)

**Q149.** What is the Result&lt;T, E&gt; pattern?

**Answer:** Errors as values: a union of { ok: true; value } and { ok: false; error }, which the caller must check before accessing .value or .error

Unlike a thrown exception, a caller can't forget to check.

**Q150.** What does handle("-5") print with parseCount?

**Answer:** failed: count cannot be negative

handle("42") prints parsed: 42; handle("abc") prints failed: "abc" is not a number.

**Q151.** What advantage does the signature Result&lt;number, ParseError&gt; give?

**Answer:** It documents every way the function can fail without reading the implementation

Callers needn't hunt for thrown exceptions.

**Q152.** When are overloaded signatures appropriate?

**Answer:** For genuinely different call shapes, like createLogger(name) and createLogger(name, level), with one implementation

The implementation signature isn't visible to callers.

### [09 · Strict Mode Best Practices](level-4/09-strict-mode-best-practices.md)

**Q153.** How many flags does strict: true bundle?

**Answer:** Eight, including noImplicitAny and strictNullChecks

Others, like noUncheckedIndexedAccess, must be opted into.

**Q154.** What does strictNullChecks do to `user.name` when user may be undefined?

**Answer:** Makes direct access without a guard a compile error

Without it, undefined is silently assignable to everything.

**Q155.** What does strictPropertyInitialization catch?

**Answer:** Class fields never assigned in the constructor

The error is "Property 'name' has no initializer…".

**Q156.** Why does scores["grace"].toFixed(0) crash even under full --strict?

**Answer:** Record&lt;string, number&gt; claims every key is a number; noUncheckedIndexedAccess isn't part of strict

At runtime the value is undefined, giving a TypeError.

### [10 · Capstone Project — Bookmark API](level-4/10-capstone-project.md)

**Q157.** Which threads does the Bookmark API capstone combine?

**Answer:** Typed Express, a generic in-memory store, Result-based validation, noUncheckedIndexedAccess with strict, and a Vitest suite

It's built, tested and run for real.

**Q158.** Why does the capstone's tsconfig add noUncheckedIndexedAccess?

**Answer:** To close the gap strict leaves, so indexed lookups are T | undefined

It's on top of strict: true.

**Q159.** Which scripts does the package.json define for the checks?

**Answer:** typecheck (tsc --noEmit) and test (vitest run)

build runs tsc and start runs node dist/index.js.

**Q160.** How does the API handle bad input?

**Answer:** Result-based validation instead of throwing

It applies the Level 4 Result pattern.

## Mixed questions across all levels

**Q161.** What does this print?

```typescript
type Shape = { kind: "circle"; r: number } | { kind: "square"; s: number };
function area(x: Shape): number {
  return x.kind === "circle" ? Math.PI * x.r ** 2 : x.s ** 2;
}
console.log(area({ kind: "circle", r: 2 }).toFixed(2), area({ kind: "square", s: 3 }));
```

**Answer:** 12.57 9

π·2² = 12.566… → 12.57, and 3² = 9. The kind field is the discriminant that narrows each branch.

**Q162.** What does this print?

```typescript
function pair<A, B>(a: A, b: B): [A, B] { return [a, b]; }
console.log(pair("age", 30));
```

**Answer:** [ 'age', 30 ]

Two type parameters keep each position's type: the result is the tuple [string, number].

**Q163.** What does this print?

```typescript
function sum(...n: number[]): number { return n.reduce((t, x) => t + x, 0); }
console.log(sum(1, 2, 3), sum());
```

**Answer:** 6 0

The rest parameter collects arguments into an array; the empty call reduces with the initial value 0.

**Q164.** What does this print?

```typescript
function power(b: number, e: number = 2): number { return b ** e; }
console.log(power(5), power(5, 3));
```

**Answer:** 25 125

The default exponent of 2 applies when it's omitted.

**Q165.** Given a function that returns Result&lt;number, string&gt; (ok → value, failure → error), this loop prints:

```typescript
function parse(s: string): Result<number, string> {
  const n = Number(s.trim());
  if (s.trim() === "") return { ok: false, error: "empty" };
  if (Number.isNaN(n)) return { ok: false, error: "nan" };
  return { ok: true, value: n };
}
for (const s of ["42", "", "x", " 7 "]) {
  const r = parse(s);
  console.log(r.ok ? r.value : r.error);
}
```

**Answer:** 42, empty, nan, 7 (one per line)

Inside the conditional, r.ok narrows r so r.value is a number or r.error a string. The trimmed " 7 " becomes 7.

**Q166.** How many times does the wrapped function actually run?

```typescript
function memoize<A extends unknown[], R>(f: (...a: A) => R) {
  const c = new Map<string, R>(); let calls = 0;
  return {
    run: (...a: A): R => {
      const k = JSON.stringify(a);
      if (c.has(k)) return c.get(k) as R;
      calls++; const v = f(...a); c.set(k, v); return v;
    },
    calls: () => calls,
  };
}
const m = memoize((n: number) => n * n);
m.run(4); m.run(4); m.run(5);
console.log(m.calls());
```

**Answer:** 2

run(4) computes once and the second call hits the cache; run(5) is a new key. Two computations.

**Q167.** What does this print?

```typescript
function* range(a: number, b: number) { for (let i = a; i < b; i++) yield i; }
function take<T>(it: Iterable<T>, n: number): T[] {
  const r: T[] = [];
  for (const x of it) { if (r.length >= n) break; r.push(x); }
  return r;
}
console.log(take(range(10, 1_000_000), 3));
```

**Answer:** [ 10, 11, 12 ] without building a million-element array

The generator produces values on demand and take stops after three.

**Q168.** This compiles under strict. What does it print at runtime?

```typescript
const scores: Record<string, number> = { ada: 100 };
console.log(scores["grace"]);
```

**Answer:** undefined

The index signature claims every string key is a number, but the key is missing. noUncheckedIndexedAccess (not part of strict) would type it number | undefined.

**Q169.** What is logged?

```typescript
const chain: Middleware[] = [
  (c, n) => { c.log.push("a"); n(); c.log.push("a2"); },
  (c, n) => { c.log.push("b"); n(); },
];
const ctx = { log: [] as string[] };
const run = (i: number): void => { const mw = chain[i]; if (mw) mw(ctx, () => run(i + 1)); };
run(0);
console.log(ctx.log.join(","));
```

**Answer:** a,b,a2

The first middleware runs until next(), the second runs fully, then control returns to push "a2".

**Q170.** What does this print?

```typescript
class A { static count = 0; constructor() { A.count++; } }
new A(); new A();
console.log(A.count);
```

**Answer:** 2

static members belong to the class, so all instances share one counter.

**Q171.** What does this print?

```typescript
function fmt(v: string | number | undefined): string {
  if (v === undefined) return "none";
  if (typeof v === "string") return v.toUpperCase();
  return v.toFixed(1);
}
console.log(fmt(undefined), fmt("ab"), fmt(2));
```

**Answer:** none AB 2.0

Each if narrows the union; the last branch is a number, so toFixed(1) gives "2.0".

**Q172.** What does `tsc` do to your types when it emits JavaScript?

**Answer:** Checks them, then erases them: nothing remains at runtime

Types exist only at compile time, so as Todo performs no runtime validation.

**Q173.** Why is `JSON.parse(text) as Task[]` not real validation?

**Answer:** A type assertion is compile-time only; a malformed file would still pass

Validate at runtime with a type guard or schema before trusting external data.

**Q174.** Which tsconfig setting is described as the most important, and what does it bundle?

**Answer:** strict: true, which bundles eight flags such as noImplicitAny and strictNullChecks

noUncheckedIndexedAccess is not part of strict and must be added separately.

**Q175.** What does `T extends keyof U` or `K extends keyof T` give in `where<K extends keyof T>(key: K, value: T[K])`?

**Answer:** The key must exist on T and the value must match that property's type

That's why users.where("active", "yes") is a compile error when active is boolean.

**Q176.** Which is the correct way to test that a function throws in Jest or Vitest?

**Answer:** expect(() =&gt; divide(1, 0)).toThrow(...), wrapping the call in a function

Otherwise the exception is thrown while evaluating the argument, before expect runs.

**Q177.** Why do tsc --noEmit and esbuild often appear together in build scripts?

**Answer:** tsc checks types but doesn't bundle; esbuild bundles fast but doesn't type-check

A typical flow is typecheck first, then build.

**Q178.** In a multi-stage Dockerfile for a TypeScript service, what does the production stage contain?

**Answer:** Production dependencies and the compiled dist, with no compiler or source

npm ci --omit=dev plus COPY --from=build /app/dist keeps the image small and reduces attack surface.

**Q179.** What does a branded type like SafeHtml = string &amp; { readonly __brand: "SafeHtml" } achieve?

**Answer:** Forces raw strings to pass through the sanitiser, at no runtime cost

The brand exists only in the type system.

## Practical coding questions

### Safe division with Result

Implement safeDivide(a, b) returning a Result: { ok: true, value } normally, and { ok: false, error: "divide by zero" } when b is 0. Then print both results for (10, 4) and (1, 0) with JSON.stringify.

Starter code:

```
type Result<T, E> =
  | { ok: true; value: T }
  | { ok: false; error: E };

function safeDivide(a: number, b: number): Result<number, string> {
  // your code
}

console.log(JSON.stringify(safeDivide(10, 4)), JSON.stringify(safeDivide(1, 0)));
```

Expected output:

```text
{"ok":true,"value":2.5} {"ok":false,"error":"divide by zero"}
```

??? success "Show a solution"

    ```
    type Result<T, E> =
      | { ok: true; value: T }
      | { ok: false; error: E };

    function safeDivide(a: number, b: number): Result<number, string> {
      if (b === 0) return { ok: false, error: "divide by zero" };
      return { ok: true, value: a / b };
    }

    console.log(JSON.stringify(safeDivide(10, 4)), JSON.stringify(safeDivide(1, 0)));
    ```

    The return type documents both outcomes and callers must narrow with result.ok before using .value. Verified by running the solution (Node's type stripping): output matches.

### Generic groupBy

Write groupBy&lt;T, K extends string&gt;(items, key) that returns a Record&lt;K, T[]&gt; grouping items by the key function. Group ["apple", "avocado", "banana", "blueberry", "cherry"] by first letter.

Starter code:

```
function groupBy<T, K extends string>(items: T[], key: (t: T) => K): Record<K, T[]> {
  // your code
}

console.log(JSON.stringify(groupBy(["apple", "avocado", "banana", "blueberry", "cherry"], (s) => s[0] as string)));
```

Expected output:

```text
{"a":["apple","avocado"],"b":["banana","blueberry"],"c":["cherry"]}
```

??? success "Show a solution"

    ```
    function groupBy<T, K extends string>(items: T[], key: (t: T) => K): Record<K, T[]> {
      const out = {} as Record<K, T[]>;
      for (const item of items) {
        const k = key(item);
        (out[k] ??= []).push(item);
      }
      return out;
    }

    console.log(JSON.stringify(groupBy(["apple", "avocado", "banana", "blueberry", "cherry"], (s) => s[0] as string)));
    ```

    The ??= operator creates each group on first use. Two generic parameters keep both the item type and the key type. Verified by running the solution: output matches.

---

More free practice: the [TypeScript Mastery Path](index.md) curriculum, the [Interview Prep Pack](https://learnmasterypath.com/interview-pack/) and the [free resume ATS check](https://learnmasterypath.com/resume/).
