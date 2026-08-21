# 07 · Build Tooling

`tsc` type-checks and emits JavaScript, but it isn't a bundler and its
plain output isn't minified — for a browser bundle or a fast dev loop,
most real projects pair TypeScript's type checker with a separate
bundler like **esbuild**. This module compares the two and shows how
they fit together.

## The source

```typescript
// mathutil.ts
export function clamp(value: number, min: number, max: number): number {
  return Math.min(Math.max(value, min), max);
}

export function sum(nums: number[]): number {
  return nums.reduce((a, b) => a + b, 0);
}

console.log(clamp(15, 0, 10));
console.log(sum([1, 2, 3, 4]));
```

## Plain `tsc`: type-checks and emits, does not bundle

```bash
tsc mathutil.ts --outDir dist-tsc --module commonjs --target es2020
node dist-tsc/mathutil.js
```

```text
10
10
```

`tsc` compiled in about 0.24s here for one tiny file — on a real project
with hundreds of files and `--watch` off, full-program type-checking is
the slow part, not emitting JS. `tsc` also does **not** bundle: each
input file becomes its own output file with `require`/`import`
statements between them, which is fine for Node but not for shipping a
single `<script>` to a browser.

## esbuild: bundles and minifies, does not type-check

```bash
npm install -D esbuild
esbuild mathutil.ts --bundle --outfile=dist.js --platform=node --minify
node dist.js
```

```text
10
10
```

```text
  dist.js  648b

⚡ Done in 8ms
```

esbuild finished in 8ms versus `tsc`'s ~240ms for the same file — the
difference gets dramatically larger on big projects because esbuild is
written in Go and skips type-checking entirely. That last point is the
catch: esbuild will happily bundle code with type errors in it. It
strips types and transpiles; it does not verify them.

## The standard combination

Because neither tool alone is both fast and safe, the common setup
runs both, for different jobs:

```json title="package.json (scripts)"
{
  "scripts": {
    "typecheck": "tsc --noEmit",
    "build": "esbuild src/index.ts --bundle --outfile=dist/bundle.js --minify --platform=node",
    "build:safe": "npm run typecheck && npm run build"
  }
}
```

`tsc --noEmit` runs purely as a type-checking gate (in CI, or a
pre-commit hook); esbuild does the actual bundling for speed. Vite,
tsup, and most modern build tools are esbuild (or a Rust equivalent)
under the hood for exactly this reason — separating "is this valid
TypeScript" from "produce fast, small JavaScript."

## `tsconfig.json` fields that matter for builds

```json title="tsconfig.json"
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "outDir": "./dist",
    "declaration": true,
    "sourceMap": true,
    "strict": true,
    "skipLibCheck": true
  }
}
```

`"moduleResolution": "bundler"` (added in TS 5.0) tells the type checker
to resolve imports the way bundlers like esbuild/Vite actually do,
rather than mimicking Node's older CommonJS resolution — mismatches
here are a frequent source of "it builds but `tsc` complains" or vice
versa. `"declaration": true` emits `.d.ts` files, which matters if
you're publishing a library others will `import` with full types.
`"skipLibCheck": true` skips type-checking `.d.ts` files in
`node_modules`, which is almost always what you want — third-party type
definitions occasionally have errors you can't fix, and checking them
adds real time to every build.

## Traps

**esbuild silently ships type errors.** `esbuild broken.ts --bundle`
succeeds and produces runnable output even if `broken.ts` has a type
error `tsc` would reject — esbuild only transpiles syntax, it never
consults the type checker. Never rely on a bundler-only build as your
safety net; always run `tsc --noEmit` in CI.

**`declaration: true` without a matching `outDir` scatters `.d.ts`
files next to your `.ts` sources**, not into `dist/` — always pair
`declaration` with an explicit `outDir` (and usually `rootDir`) so
generated files land where you expect.

**Mixing `moduleResolution: "node"` (or the old default) with a
bundler-based dev server** produces resolution mismatches for
package `exports` maps — a package that works when imported at runtime
via the bundler can still fail `tsc`'s type-check with "Cannot find
module" if the resolution strategy doesn't match what the bundler
actually does.

## Cheat sheet

| Tool | Job | Type-checks? | Bundles? |
|---|---|---|---|
| `tsc` | Compile + type-check | Yes | No |
| `tsc --noEmit` | Type-check only (CI gate) | Yes | No |
| esbuild | Fast bundle/transpile | No | Yes |
| tsup / Vite | esbuild-powered build tool with sane defaults | No (delegates to `tsc` separately) | Yes |
| `"moduleResolution": "bundler"` | Match bundler-style import resolution | — | — |

## Exercise

Add a deliberate type error to `mathutil.ts` (e.g. call
`clamp("15", 0, 10)`). Run `tsc --noEmit` and confirm it fails with a
clear error, then run `esbuild mathutil.ts --bundle --outfile=dist.js`
and confirm it succeeds anyway and `node dist.js` still executes.
Capture both outputs to prove the type-check/bundle split for yourself.
