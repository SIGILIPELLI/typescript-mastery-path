# 09 · Modules & tsconfig Deep Dive

TypeScript projects use ES modules for organizing code across files, and a
`tsconfig.json` file to control exactly how the compiler behaves.

## Exporting and importing

```typescript
// math.ts
export function add(a: number, b: number): number {
  return a + b;
}

export function subtract(a: number, b: number): number {
  return a - b;
}

export const PI = 3.14159;
```

```typescript
// main.ts
import { add, subtract, PI } from "./math";

console.log(add(2, 3));       // 5
console.log(subtract(5, 2));  // 3
console.log(PI);              // 3.14159
```

## Default exports

```typescript
// logger.ts
export default function log(message: string): void {
  console.log(`[LOG] ${message}`);
}
```

```typescript
// main.ts
import log from "./logger";   // no braces -- default imports can be named anything

log("Application started");   // [LOG] Application started
```

Named exports (`export function add`) are generally preferred over default
exports in modern TypeScript — they're easier to refactor (renaming an import
doesn't silently work with a different meaning) and better supported by
auto-import tooling.

## Re-exporting from an index file

```typescript
// shapes/circle.ts
export function circleArea(radius: number): number {
  return Math.PI * radius * radius;
}
```

```typescript
// shapes/index.ts
export * from "./circle";
export * from "./square";
```

```typescript
// main.ts
import { circleArea } from "./shapes";   // via the index.ts barrel file
```

## Setting up tsconfig.json

```bash
npx tsc --init
```

```json
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "commonjs",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "outDir": "./dist",
    "rootDir": "./src"
  },
  "include": ["src/**/*"]
}
```

## Key compilerOptions explained

| Option | Effect |
|--------|--------|
| `strict: true` | Enables all strict type-checking flags (turn this on, always) |
| `target` | Which JS version to compile down to (`ES2020`, `ES2022`, etc.) |
| `module` | Module system to emit (`commonjs` for Node, `esnext` for bundlers) |
| `outDir` | Where compiled `.js` files are written |
| `rootDir` | Where the compiler expects your `.ts` source files |
| `esModuleInterop` | Smooths over CommonJS/ES module interop quirks |
| `skipLibCheck` | Skips type-checking `.d.ts` files from dependencies (faster builds) |

`strict: true` is the single most important setting — it enables
`strictNullChecks`, `noImplicitAny`, and several other flags that catch the
overwhelming majority of real TypeScript bugs. Turning it off to "make errors
go away" defeats the entire purpose of using TypeScript.

## Compiling and running

```bash
npx tsc              # compiles according to tsconfig.json
node dist/main.js     # run the compiled output

# Or, during development, skip the manual compile step:
npx ts-node src/main.ts
```

## Cheat sheet

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

## Exercise

Create two files: `strings.ts` exporting named functions `capitalize(s:
string): string` and `reverse(s: string): string`, and `main.ts` that imports
both and prints the results of calling each on a sample string. Then create a
`tsconfig.json` with `strict: true` and confirm `npx tsc` reports no errors.
