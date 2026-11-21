---
title: "Final Quiz"
description: "Test what you have learned across all four levels of TypeScript Mastery Path with a 19-question final quiz."
---

# Final Quiz

19 questions drawn from all four levels of this course. You need **70% (14 out of 19)** to pass —
you can retake it as many times as you like. Your best result is saved in this browser.

<div id="mp-quiz-root"></div>

<script type="application/json" id="mp-quiz-data">
[
  {
    "question": "What does this print?\n```typescript\ntype Shape = { kind: \"circle\"; r: number } | { kind: \"square\"; s: number };\nfunction area(x: Shape): number {\n  return x.kind === \"circle\" ? Math.PI * x.r ** 2 : x.s ** 2;\n}\nconsole.log(area({ kind: \"circle\", r: 2 }).toFixed(2), area({ kind: \"square\", s: 3 }));\n```\n",
    "options": [
      "25.13 9",
      "12 9",
      "12.57 6",
      "12.57 9"
    ],
    "correctIndex": 3,
    "explanation": "π·2² = 12.566… → 12.57, and 3² = 9. The kind field is the discriminant that narrows each branch."
  },
  {
    "question": "What does this print?\n```typescript\nfunction pair<A, B>(a: A, b: B): [A, B] { return [a, b]; }\nconsole.log(pair(\"age\", 30));\n```\n",
    "options": [
      "[ 'age' ]",
      "[ 30, 'age' ]",
      "[ 'age', 30 ]",
      "{ a: 'age', b: 30 }"
    ],
    "correctIndex": 2,
    "explanation": "Two type parameters keep each position's type: the result is the tuple [string, number]."
  },
  {
    "question": "What does this print?\n```typescript\nfunction sum(...n: number[]): number { return n.reduce((t, x) => t + x, 0); }\nconsole.log(sum(1, 2, 3), sum());\n```\n",
    "options": [
      "6 undefined",
      "An error: sum needs arguments",
      "6 0",
      "6 NaN"
    ],
    "correctIndex": 2,
    "explanation": "The rest parameter collects arguments into an array; the empty call reduces with the initial value 0."
  },
  {
    "question": "What does this print?\n```typescript\nfunction power(b: number, e: number = 2): number { return b ** e; }\nconsole.log(power(5), power(5, 3));\n```\n",
    "options": [
      "10 15",
      "25 15",
      "25 125",
      "5 125"
    ],
    "correctIndex": 2,
    "explanation": "The default exponent of 2 applies when it's omitted."
  },
  {
    "question": "Given a function that returns Result<number, string> (ok → value, failure → error), this loop prints:\n```typescript\nfunction parse(s: string): Result<number, string> {\n  const n = Number(s.trim());\n  if (s.trim() === \"\") return { ok: false, error: \"empty\" };\n  if (Number.isNaN(n)) return { ok: false, error: \"nan\" };\n  return { ok: true, value: n };\n}\nfor (const s of [\"42\", \"\", \"x\", \" 7 \"]) {\n  const r = parse(s);\n  console.log(r.ok ? r.value : r.error);\n}\n```\n",
    "options": [
      "An error at r.value",
      "42, 0, nan, 7",
      "42, empty, nan, 7 (one per line)",
      "42, empty, nan, 7 as strings with quotes"
    ],
    "correctIndex": 2,
    "explanation": "Inside the conditional, r.ok narrows r so r.value is a number or r.error a string. The trimmed \" 7 \" becomes 7."
  },
  {
    "question": "How many times does the wrapped function actually run?\n```typescript\nfunction memoize<A extends unknown[], R>(f: (...a: A) => R) {\n  const c = new Map<string, R>(); let calls = 0;\n  return {\n    run: (...a: A): R => {\n      const k = JSON.stringify(a);\n      if (c.has(k)) return c.get(k) as R;\n      calls++; const v = f(...a); c.set(k, v); return v;\n    },\n    calls: () => calls,\n  };\n}\nconst m = memoize((n: number) => n * n);\nm.run(4); m.run(4); m.run(5);\nconsole.log(m.calls());\n```\n",
    "options": [
      "3",
      "0",
      "1",
      "2"
    ],
    "correctIndex": 3,
    "explanation": "run(4) computes once and the second call hits the cache; run(5) is a new key. Two computations."
  },
  {
    "question": "What does this print?\n```typescript\nfunction* range(a: number, b: number) { for (let i = a; i < b; i++) yield i; }\nfunction take<T>(it: Iterable<T>, n: number): T[] {\n  const r: T[] = [];\n  for (const x of it) { if (r.length >= n) break; r.push(x); }\n  return r;\n}\nconsole.log(take(range(10, 1_000_000), 3));\n```\n",
    "options": [
      "An error",
      "[ 10, 11, 12 ] without building a million-element array",
      "[ 0, 1, 2 ]",
      "[ 10, 11, 12 ] after allocating a million numbers"
    ],
    "correctIndex": 1,
    "explanation": "The generator produces values on demand and take stops after three."
  },
  {
    "question": "This compiles under strict. What does it print at runtime?\n```typescript\nconst scores: Record<string, number> = { ada: 100 };\nconsole.log(scores[\"grace\"]);\n```\n",
    "options": [
      "A compile error",
      "undefined",
      "null",
      "0"
    ],
    "correctIndex": 1,
    "explanation": "The index signature claims every string key is a number, but the key is missing. noUncheckedIndexedAccess (not part of strict) would type it number | undefined."
  },
  {
    "question": "What is logged?\n```typescript\nconst chain: Middleware[] = [\n  (c, n) => { c.log.push(\"a\"); n(); c.log.push(\"a2\"); },\n  (c, n) => { c.log.push(\"b\"); n(); },\n];\nconst ctx = { log: [] as string[] };\nconst run = (i: number): void => { const mw = chain[i]; if (mw) mw(ctx, () => run(i + 1)); };\nrun(0);\nconsole.log(ctx.log.join(\",\"));\n```\n",
    "options": [
      "b,a,a2",
      "a,b,a2",
      "a,b",
      "a,a2,b"
    ],
    "correctIndex": 1,
    "explanation": "The first middleware runs until next(), the second runs fully, then control returns to push \"a2\"."
  },
  {
    "question": "What does this print?\n```typescript\nclass A { static count = 0; constructor() { A.count++; } }\nnew A(); new A();\nconsole.log(A.count);\n```\n",
    "options": [
      "1",
      "2",
      "0",
      "undefined"
    ],
    "correctIndex": 1,
    "explanation": "static members belong to the class, so all instances share one counter."
  },
  {
    "question": "What does this print?\n```typescript\nfunction fmt(v: string | number | undefined): string {\n  if (v === undefined) return \"none\";\n  if (typeof v === \"string\") return v.toUpperCase();\n  return v.toFixed(1);\n}\nconsole.log(fmt(undefined), fmt(\"ab\"), fmt(2));\n```\n",
    "options": [
      "none AB 2.0",
      "none AB 2",
      "undefined AB 2",
      "none ab 2.0"
    ],
    "correctIndex": 0,
    "explanation": "Each if narrows the union; the last branch is a number, so toFixed(1) gives \"2.0\"."
  },
  {
    "question": "What does `tsc` do to your types when it emits JavaScript?",
    "options": [
      "Turns them into classes",
      "Keeps them as runtime checks",
      "Converts them to comments",
      "Checks them, then erases them: nothing remains at runtime"
    ],
    "correctIndex": 3,
    "explanation": "Types exist only at compile time, so as Todo performs no runtime validation."
  },
  {
    "question": "Why is `JSON.parse(text) as Task[]` not real validation?",
    "options": [
      "It is checked by strict",
      "It throws on wrong shapes",
      "A type assertion is compile-time only; a malformed file would still pass",
      "It converts the objects"
    ],
    "correctIndex": 2,
    "explanation": "Validate at runtime with a type guard or schema before trusting external data."
  },
  {
    "question": "Which tsconfig setting is described as the most important, and what does it bundle?",
    "options": [
      "strict: true, which bundles eight flags such as noImplicitAny and strictNullChecks",
      "target, which picks the JS version",
      "skipLibCheck, which hides errors",
      "outDir, which collects output"
    ],
    "correctIndex": 0,
    "explanation": "noUncheckedIndexedAccess is not part of strict and must be added separately."
  },
  {
    "question": "What does `T extends keyof U` or `K extends keyof T` give in `where<K extends keyof T>(key: K, value: T[K])`?",
    "options": [
      "Only string keys work",
      "The key must exist on T and the value must match that property's type",
      "The value can be any type",
      "Any key is allowed"
    ],
    "correctIndex": 1,
    "explanation": "That's why users.where(\"active\", \"yes\") is a compile error when active is boolean."
  },
  {
    "question": "Which is the correct way to test that a function throws in Jest or Vitest?",
    "options": [
      "expect(divide).toBe(Error)",
      "try the call outside expect",
      "expect(() => divide(1, 0)).toThrow(...), wrapping the call in a function",
      "expect(divide(1, 0)).toThrow(...)"
    ],
    "correctIndex": 2,
    "explanation": "Otherwise the exception is thrown while evaluating the argument, before expect runs."
  },
  {
    "question": "Why do tsc --noEmit and esbuild often appear together in build scripts?",
    "options": [
      "tsc is only for tests",
      "esbuild type-checks and tsc bundles",
      "tsc checks types but doesn't bundle; esbuild bundles fast but doesn't type-check",
      "They do the same job"
    ],
    "correctIndex": 2,
    "explanation": "A typical flow is typecheck first, then build."
  },
  {
    "question": "In a multi-stage Dockerfile for a TypeScript service, what does the production stage contain?",
    "options": [
      "The whole repository",
      "Only src",
      "Production dependencies and the compiled dist, with no compiler or source",
      "TypeScript and @types"
    ],
    "correctIndex": 2,
    "explanation": "npm ci --omit=dev plus COPY --from=build /app/dist keeps the image small and reduces attack surface."
  },
  {
    "question": "What does a branded type like SafeHtml = string & { readonly __brand: \"SafeHtml\" } achieve?",
    "options": [
      "Forces raw strings to pass through the sanitiser, at no runtime cost",
      "Escapes HTML automatically",
      "Checks the string at runtime",
      "Encrypts the string"
    ],
    "correctIndex": 0,
    "explanation": "The brand exists only in the type system."
  }
]
</script>

## Coding challenges

Try these on your own computer, then compare your output. Nothing you write is sent anywhere.

### Safe division with Result

Implement safeDivide(a, b) returning a Result: { ok: true, value } normally, and { ok: false, error: "divide by zero" } when b is 0. Then print both results for (10, 4) and (1, 0) with JSON.stringify.

**Starter code**

```typescript
type Result<T, E> =
  | { ok: true; value: T }
  | { ok: false; error: E };

function safeDivide(a: number, b: number): Result<number, string> {
  // your code
}

console.log(JSON.stringify(safeDivide(10, 4)), JSON.stringify(safeDivide(1, 0)));
```

**Expected output**

```text
{"ok":true,"value":2.5} {"ok":false,"error":"divide by zero"}
```

<details><summary>Show solution and explanation</summary>

<pre><code>type Result&lt;T, E&gt; =
  | { ok: true; value: T }
  | { ok: false; error: E };

function safeDivide(a: number, b: number): Result&lt;number, string&gt; {
  if (b === 0) return { ok: false, error: &quot;divide by zero&quot; };
  return { ok: true, value: a / b };
}

console.log(JSON.stringify(safeDivide(10, 4)), JSON.stringify(safeDivide(1, 0)));
</code></pre>

<p>The return type documents both outcomes and callers must narrow with result.ok before using .value. Verified by running the solution (Node&#x27;s type stripping): output matches.</p>
</details>

### Generic groupBy

Write groupBy<T, K extends string>(items, key) that returns a Record<K, T[]> grouping items by the key function. Group ["apple", "avocado", "banana", "blueberry", "cherry"] by first letter.

**Starter code**

```typescript
function groupBy<T, K extends string>(items: T[], key: (t: T) => K): Record<K, T[]> {
  // your code
}

console.log(JSON.stringify(groupBy(["apple", "avocado", "banana", "blueberry", "cherry"], (s) => s[0] as string)));
```

**Expected output**

```text
{"a":["apple","avocado"],"b":["banana","blueberry"],"c":["cherry"]}
```

<details><summary>Show solution and explanation</summary>

<pre><code>function groupBy&lt;T, K extends string&gt;(items: T[], key: (t: T) =&gt; K): Record&lt;K, T[]&gt; {
  const out = {} as Record&lt;K, T[]&gt;;
  for (const item of items) {
    const k = key(item);
    (out[k] ??= []).push(item);
  }
  return out;
}

console.log(JSON.stringify(groupBy([&quot;apple&quot;, &quot;avocado&quot;, &quot;banana&quot;, &quot;blueberry&quot;, &quot;cherry&quot;], (s) =&gt; s[0] as string)));
</code></pre>

<p>The ??= operator creates each group on first use. Two generic parameters keep both the item type and the key type. Verified by running the solution: output matches.</p>
</details>
