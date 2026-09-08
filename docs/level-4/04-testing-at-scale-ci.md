# 04 · Testing at Scale & CI

A handful of tests running locally is a different problem from
hundreds of tests running in CI across multiple Node versions with a
coverage gate. This module adds coverage reporting to the Vitest setup
from Level 3 and wires up a GitHub Actions workflow that runs it on
every push.

## Coverage with `@vitest/coverage-v8`

```bash
npm install -D @vitest/coverage-v8
```

Reusing `notifier.ts` / `notifier.test.ts` from Level 3, Module 06:

```bash
vitest run --coverage
```

```text
 RUN  v4.1.11
      Coverage enabled with v8

 Test Files  1 passed (1)
      Tests  3 passed (3)
   Duration  183ms

=============================== Coverage summary ===============================
Statements   : 100% ( 5/5 )
Branches     : 100% ( 2/2 )
Functions    : 100% ( 2/2 )
Lines        : 100% ( 5/5 )
================================================================================
```

Every statement, branch, function, and line in `notifier.ts` was
exercised by the three tests written earlier — the failure-path test
(`rejects.toThrow`) is specifically what covers the `if (!ok) throw`
branch; without it, `Branches` would read `50% (1/2)`.

## Enforcing a coverage floor

```typescript title="vitest.config.ts"
import { defineConfig } from "vitest/config";

export default defineConfig({
  test: {
    coverage: {
      provider: "v8",
      reporter: ["text", "lcov"],
      thresholds: {
        statements: 80,
        branches: 75,
        functions: 80,
        lines: 80,
      },
    },
  },
});
```

With `thresholds` set, `vitest run --coverage` exits with a non-zero
status code if actual coverage falls below any number here — this is
what turns a coverage report into an actual CI gate rather than just
information.

## GitHub Actions workflow

```yaml title=".github/workflows/ci.yml"
name: CI

on:
  push:
    branches: [main]
  pull_request:

jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        node-version: [18.x, 20.x, 22.x]
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node-version }}
          cache: "npm"
      - run: npm ci
      - run: npm run typecheck
      - run: npm run test -- --coverage
```

The `matrix.node-version` array runs the full job **three times**, once
per Node version — this catches version-specific issues (a newer
`Array` method your code relies on that doesn't exist on Node 18, for
example) that a single-version CI run would miss entirely. `npm ci`
(not `npm install`) is deliberate: it installs exactly what
`package-lock.json` specifies and fails if the lockfile is out of sync,
which is what you want in CI — reproducible installs, not
"whatever resolves today."

## Separating the type-check step from the test step

```json title="package.json (scripts)"
{
  "scripts": {
    "typecheck": "tsc --noEmit",
    "test": "vitest run"
  }
}
```

Running `typecheck` and `test` as two distinct CI steps (rather than
relying on Vitest's own TypeScript transpilation to catch type errors)
matters because **Vitest doesn't type-check your code** — it uses
esbuild under the hood (the same tool from Level 3, Module 07) purely
to strip types and run your tests fast. A test file can have a type
error in it and still pass every assertion in `vitest run`; only the
separate `tsc --noEmit` step catches that.

## Traps

**"All tests pass" and "no type errors" are two different claims that
CI must check separately.** Because Vitest transpiles with esbuild, a
CI pipeline with only a `test` step (no `typecheck` step) will
happily merge a PR containing a type error, as long as no test happens
to exercise the broken code path.

**Coverage percentage measures execution, not correctness.** 100%
statement coverage on `notifier.ts` means every line *ran* during
tests — it says nothing about whether the assertions checked the right
things. A test that calls a function and asserts nothing about its
result still counts as coverage.

**`npm install` in CI can silently change the lockfile** if
`package.json` and `package-lock.json` have drifted (e.g. a manual edit
to one but not the other), masking a real dependency problem. `npm ci`
refuses to run in that situation instead, which is exactly the
fail-loud behavior you want in a pipeline.

**A coverage `thresholds` failure exits non-zero, but only if you
actually check the exit code** — a CI step that runs the coverage
command but has `continue-on-error: true` (or equivalent) will show a
green checkmark even when the threshold isn't met.

## How It Actually Works

Running `tsc --noEmit` as a dedicated CI step (separate from your test runner and your bundler's own transpile-only pass) matters because, as covered in the build-tooling lesson, fast transpile pipelines deliberately skip full type checking — so a CI pipeline that only runs tests through a transpile-only tool and never separately invokes the real checker can pass green while shipping genuine type errors; `--noEmit` runs the full structural-checking pass across the whole program graph without writing any `.js` output, making it the one step in a typical pipeline that's actually exercising the type checker in full, rather than a per-file transpiler's stripped-down parse-and-erase.

Splitting type-checking into its own CI step (versus bundling it into a build or test command) is also a real performance decision grounded in the incremental-compilation model from the performance lesson: caching `.tsbuildinfo` between CI runs (persisting it as a build cache artifact) lets `tsc --build` skip re-verifying files whose exported type signatures haven't changed, which only pays off if that step is isolated and its cache is deliberately preserved across runs — folding type checking into a bundler step that doesn't understand or preserve `.tsbuildinfo` loses this incremental benefit entirely, turning every CI run into a full, uncached check.

Type coverage tools (measuring what fraction of expressions in a codebase resolve to `any` versus a specific type) work by walking the same checker-produced type information every IDE hover tooltip uses — they call into the TypeScript compiler API's `getTypeAtLocation` for every node in the program and count how many resolve to the `any` type specifically, which is why type-coverage percentage is a genuinely different signal from "does `tsc` report zero errors": a program riddled with `any` (silencing structural checking everywhere it appears) can still compile with zero errors, because `any` is defined to be assignable to and from everything.

## Cheat sheet

| Tool/config | Purpose |
|---|---|
| `@vitest/coverage-v8` | Adds `--coverage` reporting to Vitest |
| `coverage.thresholds` in `vitest.config.ts` | Fails the run below a coverage floor |
| Separate `typecheck` and `test` npm scripts | Vitest doesn't type-check; `tsc --noEmit` must run separately |
| `strategy.matrix.node-version` in Actions | Runs the full suite across multiple Node versions |
| `npm ci` (not `npm install`) in CI | Reproducible installs; fails on lockfile drift |

## Exercise

Add a fourth test to `notifier.test.ts` for a new `retryWelcome` method
(from the Level 3 Module 06 exercise) and run `vitest run --coverage`
before and after adding it. Record the `Branches` percentage in both
runs and explain, in a sentence, which branch the new test covers that
the original three didn't.
