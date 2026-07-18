# 01 · Setup & First Program

## Install Node.js

TypeScript compiles down to JavaScript, and that JavaScript runs on
[Node.js](https://nodejs.org). Install a recent LTS release:

```bash
# macOS (Homebrew)
brew install node

# Ubuntu/Debian
sudo apt install nodejs npm

# Windows/any OS: installer from https://nodejs.org
```

Verify the install:

```bash
node --version
# v20.x.x

npm --version
# 10.x.x
```

`node` runs JavaScript outside a browser; `npm` (Node Package Manager) installs
packages — including TypeScript itself.

## Installing TypeScript

TypeScript is distributed as an npm package. Install it globally so the `tsc`
(TypeScript Compiler) command is available everywhere, or locally per-project
(the more common real-world choice):

```bash
# Global install -- fine for learning/experimenting
npm install -g typescript

# Per-project install -- the standard for real projects
mkdir my-ts-project && cd my-ts-project
npm init -y
npm install --save-dev typescript
```

Verify:

```bash
tsc --version
# Version 5.x.x
```

With a local install, run the compiler via `npx` (which finds the
project-local binary) instead of assuming it's on your global `PATH`:

```bash
npx tsc --version
```

## Your first program

Create `hello.ts`:

```typescript
// hello.ts
function greet(name: string): string {
  return `Hello, ${name}!`;
}

console.log(greet("world"));
```

Compile it to plain JavaScript, then run the result with Node:

```bash
npx tsc hello.ts
# produces hello.js -- plain JavaScript, no types left

node hello.js
# Hello, world!
```

`tsc` **type-checks** your code and then **erases all the types**, emitting
ordinary JavaScript. Types exist only at compile time — they cost nothing at
runtime and leave no trace in the output.

## Catching a type error

Change the call to pass a number instead of a string:

```typescript
console.log(greet(42));
```

```bash
npx tsc hello.ts
# hello.ts:6:19 - error TS2345: Argument of type 'number' is not assignable
# to parameter of type 'string'.
```

`tsc` refuses to compile until the error is fixed (unless you explicitly tell
it not to) — this is the whole point of TypeScript: catching this class of bug
before the program ever runs, instead of discovering it at 2am in production.

## `tsconfig.json` basics

Real projects don't call `tsc` with a filename directly — they use a
`tsconfig.json` that describes the whole project: which files to include, what
JavaScript to target, and how strict to be.

```bash
npx tsc --init
```

This generates a `tsconfig.json` with many options commented out. A sensible
starting point for a new project:

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "commonjs",
    "rootDir": "./src",
    "outDir": "./dist",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true
  },
  "include": ["src/**/*"]
}
```

| Option | Meaning |
|--------|---------|
| `target` | Which JavaScript version to emit (e.g. `ES2022`) |
| `module` | Module system for the output (`commonjs` for Node, `esnext` for bundlers) |
| `rootDir` / `outDir` | Where source lives, where compiled output goes |
| `strict` | Turns on the full set of strict type-checking flags (always keep this on — see [Module 9](09-modules-tsconfig.md)) |
| `include` | Which files/folders `tsc` should compile |

With a `tsconfig.json` in place, just run `tsc` with no arguments from the
project root — it finds the config automatically:

```bash
mkdir src
mv hello.ts src/
npx tsc
# compiles src/hello.ts -> dist/hello.js, following tsconfig.json
```

## Running TypeScript directly with ts-node

For quick scripts and learning, recompiling and then running a separate `.js`
file is friction you don't need. [`ts-node`](https://typestrong.org/ts-node/)
compiles and runs TypeScript in one step:

```bash
npm install --save-dev ts-node

npx ts-node src/hello.ts
# Hello, world!
```

`ts-node` is great for local development and this course's exercises. Compiled
`tsc` output is still what you'd deploy to production, since `ts-node`
recompiles on every run and carries extra startup overhead.

## Choosing an editor

**VS Code** (free, from Microsoft) has first-class TypeScript support built
in — inline error squiggles, autocomplete, and refactoring all work out of the
box with zero configuration, since VS Code and TypeScript share the same
underlying language service. Any other editor with a TypeScript plugin
(WebStorm, Neovim + a language server) works too — pick one and move on, since
the type system matters far more than the editor.

## Exercise

Set up a fresh project: run `npm init -y`, install `typescript` and
`ts-node` as dev dependencies, and generate a `tsconfig.json` with `tsc
--init`. Edit it to set `"strict": true`, `"rootDir": "./src"`, and
`"outDir": "./dist"`. Then write `src/greeter.ts` with a function that prints
a greeting for three different names, and run it directly with
`npx ts-node src/greeter.ts`. Finally compile it with `npx tsc` and run the
emitted JavaScript with `node dist/greeter.js` to confirm both paths work.
