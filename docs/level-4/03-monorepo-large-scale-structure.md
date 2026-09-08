# 03 · Monorepo & Large-Scale Structure

A monorepo with multiple TypeScript packages needs two separate things
to work: `tsc` needs to know how packages depend on each other for
**type-checking**, and Node needs to be able to actually **resolve**
`import { x } from "@acme/core"` at runtime. TypeScript's project
references solve the first; a package manager's workspace linking
solves the second — and conflating them is the most common monorepo
setup mistake.

## Layout

```text
mono/
├── tsconfig.json                (solution file — no code of its own)
└── packages/
    ├── core/
    │   ├── tsconfig.json
    │   └── src/index.ts
    └── api/
        ├── tsconfig.json
        └── src/index.ts
```

## `packages/core` — a leaf package

```typescript title="packages/core/src/index.ts"
export interface User {
  id: number;
  name: string;
}

export function formatUser(u: User): string {
  return `#${u.id} ${u.name}`;
}
```

```json title="packages/core/tsconfig.json"
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "commonjs",
    "declaration": true,
    "composite": true,
    "outDir": "./dist",
    "rootDir": "./src",
    "strict": true
  },
  "include": ["src"]
}
```

`"composite": true` is required on every package a *reference* points
at — it forces `declaration: true` and stricter incremental-build
bookkeeping so other packages can depend on this one's compiled
`.d.ts` output.

## `packages/api` — depends on `core`

```typescript title="packages/api/src/index.ts"
import { formatUser, User } from "@acme/core";

const u: User = { id: 1, name: "Ada" };
console.log(formatUser(u));
```

```json title="packages/api/tsconfig.json"
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "commonjs",
    "composite": true,
    "outDir": "./dist",
    "rootDir": "./src",
    "strict": true,
    "paths": {
      "@acme/core": ["../core/src"]
    }
  },
  "references": [{ "path": "../core" }],
  "include": ["src"]
}
```

`references` tells `tsc --build` about the dependency graph (build
`core` before `api`, rebuild `api` if `core` changes). `paths` tells the
**type checker** where to resolve `@acme/core` from during editing and
type-checking — it does not affect what Node does at runtime.

## The root "solution" file

```json title="tsconfig.json"
{
  "files": [],
  "references": [
    { "path": "./packages/core" },
    { "path": "./packages/api" }
  ]
}
```

An empty `files: []` array plus `references` makes this a pure solution
file — running `tsc --build` from here builds every referenced package
in dependency order.

## Running it

```bash
tsc --build --verbose
```

```text
Projects in this build:
    * packages/core/tsconfig.json
    * packages/api/tsconfig.json
    * tsconfig.json

Project 'packages/core/tsconfig.json' is out of date because output
file 'packages/core/tsconfig.tsbuildinfo' does not exist
Building project '.../packages/core/tsconfig.json'...

Project 'packages/api/tsconfig.json' is out of date because output
file 'packages/api/tsconfig.tsbuildinfo' does not exist
Building project '.../packages/api/tsconfig.json'...
```

`core` builds before `api`, exactly matching the reference order.
Running `tsc --build --verbose` again with no source changes:

```text
Project 'packages/core/tsconfig.json' is up to date because newest
input 'packages/core/src/index.ts' is older than output
'packages/core/tsconfig.tsbuildinfo'

Project 'packages/api/tsconfig.json' is up to date because newest
input 'packages/api/src/index.ts' is older than output
'packages/api/tsconfig.tsbuildinfo'
```

Nothing recompiles — this is the actual mechanism behind fast CI builds
on large monorepos: `tsc --build` skips any package whose inputs
haven't changed since its last successful build.

## The trap: `tsc --build` succeeds, `node` fails

```bash
node packages/api/dist/index.js
```

```text
Error: Cannot find module '@acme/core'
Require stack:
- .../packages/api/dist/index.js
```

Type-checking passed completely — `tsc --build` reported no errors —
but the compiled JavaScript still does `require("@acme/core")`, and
plain Node has no idea what that package specifier means. `paths` is a
**compiler-only** resolution hint; it never rewrites the emitted
`require`/`import` calls. Real projects fix this with a package
manager's workspace linking:

```json title="package.json (root)"
{
  "workspaces": ["packages/*"]
}
```

```bash
npm install
```

`npm install` (or `pnpm`/`yarn` workspaces) symlinks
`node_modules/@acme/core` to `packages/core`, so the *same* unmodified
`require("@acme/core")` in the compiled output resolves at runtime too
— at that point the compiler's `paths` and the runtime's actual module
resolution agree.

## Traps

**`paths` and workspace linking solving the same-looking problem at two
different layers is the single biggest monorepo confusion.** A build
that type-checks clean but crashes on `node dist/index.js` with
`Cannot find module` almost always means `paths` was configured but
workspace linking (or a build step that rewrites import paths, like
`tsc-alias`) was not.

**Forgetting `"composite": true` on a referenced package** produces
`Referenced project '...' may not disable emit` or similar — every
project pointed at by another project's `references` array must be
composite.

**Circular references between packages are rejected outright** — `tsc
--build` refuses to build a graph where package A references B and B
references A, unlike some bundlers that tolerate certain circular
imports at runtime.

## How It Actually Works

Project references (introduced in the performance lesson) are what make a monorepo's cross-package type checking scale: without them, a single `tsc` invocation over a monorepo treats every file reachable from your entry points as one flat program, re-resolving and re-checking the *entire* dependency graph's types on every build; with `composite: true` and `references`, each package is compiled and checked as an independent unit that emits its own `.d.ts`, and a downstream package's imports resolve against that upstream `.d.ts` output rather than re-walking upstream source — meaning `tsc --build` can skip recompiling any referenced package whose inputs (tracked via each package's own `.tsbuildinfo`) haven't changed, which is a fundamentally different scaling model from a single flat check.

Path aliases (`paths` in `tsconfig.json`, mapping `@myorg/utils` to a local package path) are a **compile-time-only** resolution hint that only affects how the *checker* finds types — the emitted JS still contains the literal, unresolved import specifier (`import { x } from "@myorg/utils"`), so at runtime, whatever actually executes the compiled output (Node, a bundler) needs its own, separately-configured resolution for the same alias (a bundler's own alias config, or a workspace tool's symlinked `node_modules` entries) — a path alias that only exists in `tsconfig.json` and nowhere else compiles clean and then throws a module-not-found error the instant you run it.

Workspace tools (npm/pnpm/yarn workspaces) symlink each internal package into a shared `node_modules`, which is what lets ordinary `node`-strategy module resolution find internal packages using the same algorithm as external npm dependencies — the type checker doesn't need to know a dependency is "local" at all; it just resolves the `.d.ts` through the symlink like any other package, which is why a broken or missing workspace symlink produces exactly the same "cannot find module" checker error as a genuinely missing external dependency.

## Cheat sheet

| Concern | Solved by |
|---|---|
| "Which package depends on which" for `tsc --build` | `references` in each `tsconfig.json` |
| Compiler resolving `@acme/core` while editing/type-checking | `paths` |
| Node resolving `@acme/core` at runtime | Workspace linking (npm/pnpm/yarn workspaces) |
| Skipping unchanged packages on rebuild | `composite: true` + `.tsbuildinfo` per package |
| A package other packages can reference | `"composite": true`, `"declaration": true` |

## Exercise

Add `"workspaces": ["packages/*"]` to a root `package.json`, run
`npm install`, and re-run `node packages/api/dist/index.js` to confirm
it now prints `#1 Ada` instead of failing. Then add a third package,
`packages/utils`, that `core` itself depends on, wire up the
`references`/`paths` chain three levels deep, and confirm
`tsc --build --verbose` builds all three in the correct order.
