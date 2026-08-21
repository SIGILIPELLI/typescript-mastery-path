# 04 · Type-Safe Database Access

Real projects use Prisma, Drizzle, or Knex for this, but the underlying
techniques — generic repository classes, `keyof`-constrained queries,
and typed tagged templates for raw SQL — are worth building by hand
once so the ORM's types stop feeling like magic. This module implements
a small in-memory `Table<T>` that mirrors what those libraries generate.

## A generic table/repository

```typescript
interface Row {
  id: number;
  [key: string]: unknown;
}

class Table<T extends Row> {
  private rows: T[] = [];
  private nextId = 1;

  insert(data: Omit<T, "id">): T {
    const row = { ...data, id: this.nextId++ } as T;
    this.rows.push(row);
    return row;
  }

  findById(id: number): T | undefined {
    return this.rows.find((r) => r.id === id);
  }

  where<K extends keyof T>(key: K, value: T[K]): T[] {
    return this.rows.filter((r) => r[key] === value);
  }

  update(id: number, patch: Partial<Omit<T, "id">>): T | undefined {
    const row = this.findById(id);
    if (!row) return undefined;
    Object.assign(row, patch);
    return row;
  }

  all(): T[] {
    return [...this.rows];
  }
}

interface UserRow extends Row {
  id: number;
  name: string;
  email: string;
  active: boolean;
}

const users = new Table<UserRow>();
users.insert({ name: "Ada", email: "ada@example.com", active: true });
users.insert({ name: "Grace", email: "grace@example.com", active: false });

console.log(users.all());
console.log(users.where("active", true));
console.log(users.update(1, { email: "ada@newmail.com" }));
console.log(users.findById(99));
```

```text
[
  { name: 'Ada', email: 'ada@example.com', active: true, id: 1 },
  { name: 'Grace', email: 'grace@example.com', active: false, id: 2 }
]
[ { name: 'Ada', email: 'ada@example.com', active: true, id: 1 } ]
{ name: 'Ada', email: 'ada@newmail.com', active: true, id: 1 }
undefined
```

`where<K extends keyof T>(key: K, value: T[K])` is the important line:
`key` must be an actual property of `T`, and `value` must match *that
property's* type — `users.where("active", "yes")` is a compile error
because `active` is `boolean`, not `string`. This is exactly the trick
Prisma and Drizzle use to type-check `.where({ active: true })` calls
against your schema.

## A typed tagged template for raw SQL

```typescript
function sql<T>(
  strings: TemplateStringsArray,
  ...values: unknown[]
): { text: string; values: unknown[] } {
  return { text: strings.join("?"), values };
}

interface UserQueryResult {
  id: number;
  name: string;
}

const query = sql<UserQueryResult>`SELECT id, name FROM users WHERE active = ${true}`;
console.log(query);
```

```text
{
  text: 'SELECT id, name FROM users WHERE active = ?',
  values: [ true ]
}
```

`sql<UserQueryResult>` doesn't change what runs — it exists purely so a
caller (or a query-runner function) can say "the rows this query
produces have this shape," letting the *result* of running `query` be
typed as `UserQueryResult[]` even though the template itself only
builds a string.

## Traps

**A generic parameter bound to `Record<string, unknown>` rejects normal
interfaces.** Trying `function sql<T extends Record<string, unknown>>`
and then calling `sql<UserQueryResult>` fails:

```text
error TS2344: Type 'UserQueryResult' does not satisfy the constraint
'Record<string, unknown>'. Index signature for type 'string' is
missing in type 'UserQueryResult'.
```

A plain `interface { id: number; name: string }` has no index
signature, so it isn't assignable to `Record<string, unknown>` even
though every property it has fits. Either add an index signature to the
interface, use `Pick<T, keyof T>`-style mapped constraints, or — usually
simplest — drop the constraint entirely and let `T` be inferred from
usage, as done above.

**`Object.assign(row, patch)` bypasses type checking on individual
fields** — `row` is `T`, and `Object.assign`'s overloads widen the
result type, so a patch with a wrong-shaped value can slip through
without a compile error. Prefer looping and assigning per-key with
`Partial<T>[K]` typed access if you need per-field safety.

**`as T` in `insert()` is a real cast, not a guarantee.** If `Omit<T, "id">`
doesn't actually match what the caller passed (e.g. extra required
fields your `interface` declares elsewhere), TypeScript won't catch it
here — the cast tells the compiler to trust you.

## Cheat sheet

| Pattern | What it buys you |
|---|---|
| `class Table<T extends Row>` | One repository implementation, reused per entity |
| `where<K extends keyof T>(key: K, value: T[K])` | Column name and value type stay linked |
| `Omit<T, "id">` for inserts | Callers can't (and don't need to) supply an id |
| `Partial<Omit<T, "id">>` for updates | Any subset of non-id fields, nothing else |
| `sql<T>` tagged template | Attaches a result type to a query without changing its runtime shape |

## Exercise

Add a `delete(id: number): boolean` method to `Table<T>` that removes a
row and returns whether it existed, then add a `count<K extends keyof T>(key: K, value: T[K]): number`
method built on top of `where`. Compile with `--strict` and run a small
script proving `count("active", true)` and `count("active", false)`
return the right numbers before and after a `delete`.
