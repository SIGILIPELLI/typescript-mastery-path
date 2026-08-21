# 07 · Performance Optimization

TypeScript compiles away to plain JavaScript, so runtime performance
techniques here are really JavaScript performance techniques — but
typing them well matters: a generic `memoize<Args, Result>` or a typed
object pool needs to preserve the exact call signature and result type
of whatever it wraps, or it becomes a liability instead of a help. This
module covers memoization, lazy evaluation, and object pooling with
full type safety.

## Memoization with a typed cache

```typescript
function memoize<Args extends unknown[], Result>(
  fn: (...args: Args) => Result,
  keyFn: (...args: Args) => string = (...args) => JSON.stringify(args)
): (...args: Args) => Result {
  const cache = new Map<string, Result>();
  return (...args: Args): Result => {
    const key = keyFn(...args);
    if (cache.has(key)) {
      return cache.get(key) as Result;
    }
    const result = fn(...args);
    cache.set(key, result);
    return result;
  };
}

function slowSquare(n: number): number {
  let total = 0;
  for (let i = 0; i < 5_000_000; i++) total += 1;
  return n * n + (total - 5_000_000);
}

const fastSquare = memoize(slowSquare);

console.time("cold");
fastSquare(7);
console.timeEnd("cold");

console.time("warm");
fastSquare(7);
console.timeEnd("warm");
```

```text
cold: 3.284ms
warm: 0.004ms
```

The cold call does the full 5-million-iteration loop; the warm call
hits the `Map` cache and returns immediately — roughly 800x faster in
this run. `memoize<Args extends unknown[], Result>` is generic over
both the parameter tuple and return type, so `fastSquare` keeps
`slowSquare`'s exact signature `(n: number) => number` — callers get no
indication anything changed except speed.

## Lazy evaluation with generators

```typescript
function* lazyRange(start: number, end: number): Generator<number> {
  for (let i = start; i < end; i++) {
    yield i;
  }
}

function take<T>(iter: Iterable<T>, n: number): T[] {
  const result: T[] = [];
  for (const item of iter) {
    if (result.length >= n) break;
    result.push(item);
  }
  return result;
}

console.log(take(lazyRange(0, 1_000_000), 5));
```

```text
[ 0, 1, 2, 3, 4 ]
```

`lazyRange(0, 1_000_000)` doesn't allocate a million-element array —
each value is produced on demand as `take` iterates, and `take` stops
pulling after 5 values. Compare this to `Array.from({length: 1_000_000})`
followed by `.slice(0, 5)`, which does the full allocation regardless of
how many values are actually needed.

## Object pooling to reduce GC pressure

```typescript
class Vector {
  x = 0;
  y = 0;
  reset(x: number, y: number): this {
    this.x = x;
    this.y = y;
    return this;
  }
}

class Pool<T> {
  private items: T[] = [];
  constructor(private factory: () => T) {}
  acquire(): T {
    return this.items.pop() ?? this.factory();
  }
  release(item: T): void {
    this.items.push(item);
  }
}

const pool = new Pool(() => new Vector());
const v1 = pool.acquire().reset(1, 2);
pool.release(v1);
const v2 = pool.acquire().reset(3, 4);
console.log(v2, v2 === v1);
```

```text
Vector { x: 3, y: 4 } true
```

`v2 === v1` is `true` — `acquire()` reused the exact same `Vector`
instance instead of allocating a new one, because it had just been
`release`d back into the pool. This pattern matters in hot loops
(game loops, real-time data processing) where allocating and
garbage-collecting thousands of short-lived objects per second causes
visible GC pauses; `Pool<T>` being generic means the same pooling logic
works for any poolable type, not just `Vector`.

## Traps

**`Iterable<T>` iteration requires a modern target.** Compiling
`for (const item of iter)` over a generic `Iterable<T>` against an old
`--target` (below ES2015) fails with:

```text
error TS2802: Type 'Iterable<T>' can only be iterated through when
using the '--downlevelIteration' flag or with a '--target' of 'es2015'
or higher.
```

This is a compile-time-only concern — the fix is `--target es2020` (or
higher) or `--downlevelIteration`, not a runtime change — but it
surprises people because the same code works fine in a plain `.js` file
or in an editor with looser settings.

**A memoization cache with no eviction is a memory leak in disguise.**
The `Map` in `memoize` above grows forever — fine for a pure function
called with a small, bounded set of inputs (like `slowSquare(7)`
repeatedly), a real problem for a function called with unbounded or
user-controlled input. A production memoizer usually needs an LRU cache
or a max-size cutoff.

**Returned pooled objects retain stale state until explicitly `reset`.**
`pool.acquire()` alone (without calling `.reset(...)`) hands back
whatever `x`/`y` the object had from its *previous* use — a caller that
forgets to reset introduces a hard-to-trace bug where an object
"remembers" values from a completely unrelated earlier use.

**`JSON.stringify(args)` as the default memoization key breaks for
non-serializable arguments** (functions, `undefined` inside objects,
circular references) — pass an explicit `keyFn` for any `memoize` call
where the arguments aren't plain, simple values.

## Cheat sheet

| Technique | Solves | Typed via |
|---|---|---|
| `memoize<Args, Result>` | Repeated calls with the same expensive arguments | Generic over parameter tuple and return type |
| Generators (`function*`) | Avoiding full-array allocation for large/infinite sequences | `Generator<T>` / `Iterable<T>` |
| `Pool<T>` | GC churn from many short-lived objects in hot loops | Generic factory + typed acquire/release |
| `--target es2020`+ | Enables direct `for...of` over generic `Iterable<T>` | Compiler flag, not a code change |

## Exercise

Add an LRU eviction policy to `memoize`: cap the cache at `maxSize`
entries, and when a new key is added past that cap, evict the
least-recently-used entry (a `Map` preserves insertion order, so
re-inserting a key on cache hit — delete then set — is enough to track
recency). Prove it works by memoizing a function, calling it with 3
different arguments against a `maxSize` of 2, and logging which cache
entries survive.
