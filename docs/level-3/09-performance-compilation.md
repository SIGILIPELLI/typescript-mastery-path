# 09 · Performance & Compilation

TypeScript's compiler is doing real work — parsing, binding, and a type
checker that can recursively expand generic types — and on a large
project that work is visible as slow `tsc` runs and a laggy editor. This
module uses `--extendedDiagnostics` to actually measure where the time
goes and shows the two biggest levers for speeding it up.

## Measuring with `--extendedDiagnostics`

```typescript
// heavy.ts
type DeepPartial<T> = T extends object
  ? { [K in keyof T]?: DeepPartial<T[K]> }
  : T;

interface Big {
  a: { b: { c: { d: { e: number; f: string } } } };
}

const patch: DeepPartial<Big> = { a: { b: { c: { d: { e: 1 } } } } };
console.log(patch);
```

```bash
tsc --noEmit --extendedDiagnostics heavy.ts
```

```text
Files:              83
Lines:           58456
Identifiers:     49746
Symbols:         59809
Types:           35374
Instantiations:  34321
Memory used:    64902K
Check time:     0.132s
Total time:     0.158s
```

Notice **83 files** and **58,456 lines** for one small `.ts` file —
that's `lib.d.ts` (the built-in JS/DOM type definitions) being pulled
in and fully type-checked by default. `Check time` (0.132s) dominates
`Total time` here, and check time is what scales with the number and
complexity of types your code touches, not raw line count of *your*
code.

## `skipLibCheck`: the single biggest lever

```bash
tsc --noEmit --extendedDiagnostics --skipLibCheck heavy.ts
```

```text
Symbols:         32873
Types:             395
Instantiations:     91
Memory used:    25623K
Check time:     0.001s
Total time:     0.026s
```

Same file, same output — but `Check time` drops from `0.132s` to
`0.001s` and `Types` drops from 35,374 to 395. `skipLibCheck` tells
`tsc` to trust that `.d.ts` files (both `lib.d.ts` and everything in
`node_modules/@types`) are already valid and skip re-checking them —
your own code is still fully checked. This is why `skipLibCheck: true`
is in nearly every real-world `tsconfig.json`: it costs you almost
nothing in safety (you're not going to fix a bug in `lib.d.ts` anyway)
and buys back most of the checker's time on any project with a
non-trivial `node_modules`.

## Incremental builds

```json title="tsconfig.json"
{
  "compilerOptions": {
    "incremental": true,
    "tsBuildInfoFile": "./.tsbuildinfo"
  }
}
```

With `incremental: true`, `tsc` writes a `.tsbuildinfo` file recording
what it checked; the next run reads it and only re-checks files that
changed (or that depend on a changed file) instead of the whole program.
For editor responsiveness, the TypeScript language server does this
automatically — `.tsbuildinfo` matters specifically for repeated CLI/CI
invocations of `tsc`.

## Project references for large monorepos

```json title="tsconfig.json (root)"
{
  "references": [
    { "path": "./packages/core" },
    { "path": "./packages/api" }
  ]
}
```

```bash
tsc --build
```

Project references split a large codebase into smaller, independently
type-checked units with declared dependencies between them. `tsc --build`
only rebuilds a package (and its dependents) when that package's own
files change — a change deep in `packages/api` doesn't force
re-checking `packages/core` if `core` doesn't depend on `api`. This is
the mechanism, not just a convention: `tsc` enforces the reference graph
and refuses circular references.

## Traps

**Recursive conditional/mapped types like `DeepPartial<T>` cost more the
deeper the type nests**, and TypeScript enforces a hard recursion depth
limit — past it you get `error TS2589: Type instantiation is excessively
deep and possibly infinite`, even for a type that would terminate given
enough steps. Deeply generic utility types are a common source of this
in real codebases that lean hard on type-level programming (see the
Level 4 module on advanced type-level programming).

**`skipLibCheck` doesn't skip checking your own `.d.ts` files against
your own code that consumes them** — it skips checking the *internal*
correctness of library `.d.ts` files. If your code misuses a type from
a library, that error still surfaces; `skipLibCheck` only stops
`tsc` from also verifying the library's declaration file is internally
self-consistent.

**Turning on `incremental` without gitignoring `.tsbuildinfo`** pollutes
diffs with a binary-ish cache file that changes on every build — add it
to `.gitignore`.

**Editor slowness and `tsc` CLI slowness have different causes.** The
language server (what your editor uses) does incremental, single-file
reanalysis by default regardless of your `tsconfig`; a slow editor is
more often caused by an enormous single file, a very large union type,
or a plugin, not by the same things that make a cold `tsc --build` slow.

## Cheat sheet

| Setting/flag | Effect |
|---|---|
| `--extendedDiagnostics` | Prints parse/bind/check/emit timing breakdown |
| `skipLibCheck: true` | Skips re-checking `.d.ts` files; usually the biggest win |
| `incremental: true` | Caches check results in `.tsbuildinfo` between CLI runs |
| Project references + `tsc --build` | Per-package incremental checking in a monorepo |
| `TS2589` | Recursion depth limit hit on a conditional/mapped type |

## Exercise

Take the `DeepPartial<T>` example, nest `Big` two more levels deeper,
and re-run `tsc --noEmit --extendedDiagnostics` both with and without
`--skipLibCheck`. Record the four `Check time` numbers (with/without,
shallow/deep) in a short table and note which factor — lib checking or
nesting depth — moved the number more.
