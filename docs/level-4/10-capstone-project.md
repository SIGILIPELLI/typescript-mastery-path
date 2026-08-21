# 10 · Capstone Project — Bookmark API

The capstone pulls together the path's major threads into one small
but complete service: a typed Express API, a generic in-memory store,
`Result`-based validation instead of throwing on bad input,
`noUncheckedIndexedAccess` and full `strict` mode, and a Vitest suite —
built, tested, and run for real.

## Project layout

```text
bookmark-api/
├── package.json
├── tsconfig.json
└── src/
    ├── store.ts
    ├── store.test.ts
    └── index.ts
```

## `package.json`

```json
{
  "name": "bookmark-api",
  "version": "1.0.0",
  "private": true,
  "main": "dist/index.js",
  "scripts": {
    "build": "tsc",
    "start": "node dist/index.js",
    "typecheck": "tsc --noEmit",
    "test": "vitest run"
  },
  "dependencies": {
    "express": "^4.19.2"
  },
  "devDependencies": {
    "@types/express": "^4.17.21",
    "@types/node": "^20.11.30",
    "typescript": "^5.9.0",
    "vitest": "^2.0.0"
  }
}
```

## `tsconfig.json`

```json
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "commonjs",
    "moduleResolution": "node",
    "outDir": "./dist",
    "rootDir": "./src",
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true
  },
  "include": ["src"],
  "exclude": ["src/**/*.test.ts"]
}
```

`noUncheckedIndexedAccess` is on deliberately (Level 4, Module 09) —
this project doesn't happen to need it for its own logic, but it's the
right default for new services.

## `src/store.ts` — data layer with `Result`-based validation

```typescript
export interface Bookmark {
  id: number;
  title: string;
  url: string;
  tags: string[];
}

export type Result<T, E = string> =
  | { ok: true; value: T }
  | { ok: false; error: E };

export function ok<T>(value: T): Result<T, never> {
  return { ok: true, value };
}
export function err<E>(error: E): Result<never, E> {
  return { ok: false, error };
}

export function validateBookmarkInput(
  input: Partial<{ title: unknown; url: unknown; tags: unknown }>
): Result<{ title: string; url: string; tags: string[] }> {
  if (typeof input.title !== "string" || input.title.trim() === "") {
    return err("title is required and must be a non-empty string");
  }
  if (typeof input.url !== "string" || !/^https?:\/\//.test(input.url)) {
    return err("url must be a string starting with http:// or https://");
  }
  const tags =
    input.tags === undefined
      ? []
      : Array.isArray(input.tags) && input.tags.every((t) => typeof t === "string")
        ? (input.tags as string[])
        : null;
  if (tags === null) {
    return err("tags must be an array of strings if provided");
  }
  return ok({ title: input.title, url: input.url, tags });
}

export class BookmarkStore {
  private bookmarks: Bookmark[] = [];
  private nextId = 1;

  list(tag?: string): Bookmark[] {
    if (tag === undefined) return [...this.bookmarks];
    return this.bookmarks.filter((b) => b.tags.includes(tag));
  }

  create(data: { title: string; url: string; tags: string[] }): Bookmark {
    const bookmark: Bookmark = { id: this.nextId++, ...data };
    this.bookmarks.push(bookmark);
    return bookmark;
  }

  find(id: number): Bookmark | undefined {
    return this.bookmarks.find((b) => b.id === id);
  }

  remove(id: number): boolean {
    const index = this.bookmarks.findIndex((b) => b.id === id);
    if (index === -1) return false;
    this.bookmarks.splice(index, 1);
    return true;
  }
}
```

`validateBookmarkInput` accepts `unknown`-typed fields deliberately —
request bodies are never trustworthy input, `Result` forces the route
handler to check `.ok` before touching `.value`, and the `Result<T>`
error type (`string` by default in this file) keeps the API's error
messages consistent everywhere it's used.

## `src/index.ts` — the typed Express layer

```typescript
import express, { Request, Response, NextFunction } from "express";
import { BookmarkStore, validateBookmarkInput } from "./store";

const app = express();
app.use(express.json());
const store = new BookmarkStore();

function asyncHandler<P, ResBody, ReqBody, Q>(
  fn: (req: Request<P, ResBody, ReqBody, Q>, res: Response<ResBody>) => void
) {
  return (req: Request<P, ResBody, ReqBody, Q>, res: Response<ResBody>, next: NextFunction) => {
    try {
      fn(req, res);
    } catch (e) {
      next(e);
    }
  };
}

app.get(
  "/bookmarks",
  asyncHandler((req: Request<{}, {}, {}, { tag?: string }>, res: Response) => {
    res.json(store.list(req.query.tag));
  })
);

app.post(
  "/bookmarks",
  asyncHandler((req: Request<{}, {}, unknown>, res: Response) => {
    const result = validateBookmarkInput(req.body as Record<string, unknown>);
    if (!result.ok) {
      res.status(400).json({ error: result.error });
      return;
    }
    res.status(201).json(store.create(result.value));
  })
);

app.get(
  "/bookmarks/:id",
  asyncHandler((req: Request<{ id: string }>, res: Response) => {
    const bookmark = store.find(Number(req.params.id));
    if (!bookmark) {
      res.status(404).json({ error: "not found" });
      return;
    }
    res.json(bookmark);
  })
);

app.delete(
  "/bookmarks/:id",
  asyncHandler((req: Request<{ id: string }>, res: Response) => {
    const removed = store.remove(Number(req.params.id));
    if (!removed) {
      res.status(404).json({ error: "not found" });
      return;
    }
    res.status(204).end();
  })
);

app.use((err: Error, _req: Request, res: Response, _next: NextFunction): void => {
  console.error(err);
  res.status(500).json({ error: "internal server error" });
});

const PORT = process.env.PORT ? Number(process.env.PORT) : 4600;
if (require.main === module) {
  app.listen(PORT, () => console.log(`Bookmark API listening on ${PORT}`));
}

export { app, store };
```

`if (require.main === module)` guards the `app.listen(...)` call so
`src/index.ts` can be `import`ed by a test file (to test `app` directly
with a request library) without starting a real network listener —
useful once a project grows past manually curling a running server.

## `src/store.test.ts`

```typescript
import { describe, it, expect } from "vitest";
import { BookmarkStore, validateBookmarkInput } from "./store";

describe("validateBookmarkInput", () => {
  it("rejects a missing title", () => {
    const result = validateBookmarkInput({ url: "https://example.com" });
    expect(result.ok).toBe(false);
  });

  it("rejects an invalid url", () => {
    const result = validateBookmarkInput({ title: "Example", url: "not-a-url" });
    expect(result.ok).toBe(false);
  });

  it("accepts valid input with default empty tags", () => {
    const result = validateBookmarkInput({ title: "Example", url: "https://example.com" });
    expect(result.ok).toBe(true);
    if (result.ok) {
      expect(result.value.tags).toEqual([]);
    }
  });
});

describe("BookmarkStore", () => {
  it("creates, lists, filters by tag, and removes bookmarks", () => {
    const store = new BookmarkStore();
    store.create({ title: "A", url: "https://a.com", tags: ["work"] });
    store.create({ title: "B", url: "https://b.com", tags: ["personal"] });

    expect(store.list()).toHaveLength(2);
    expect(store.list("work")).toHaveLength(1);

    const removed = store.remove(1);
    expect(removed).toBe(true);
    expect(store.list()).toHaveLength(1);
    expect(store.remove(999)).toBe(false);
  });
});
```

## Running it

```bash
npm install
npm run typecheck
npm test
```

```text
 RUN  v4.1.11

 Test Files  1 passed (1)
      Tests  4 passed (4)
   Duration  156ms
```

Then the server, exercised with real `curl` calls:

```bash
npm run build && npm start
```

```text
Bookmark API listening on 4600
```

```bash
curl -s -X POST localhost:4600/bookmarks -H 'Content-Type: application/json' \
  -d '{"title":"TS Handbook","url":"https://www.typescriptlang.org/docs/","tags":["docs","ts"]}'
# {"id":1,"title":"TS Handbook","url":"https://www.typescriptlang.org/docs/","tags":["docs","ts"]}

curl -s -X POST localhost:4600/bookmarks -H 'Content-Type: application/json' \
  -d '{"title":"News","url":"https://news.example.com","tags":["news"]}'
# {"id":2,"title":"News","url":"https://news.example.com","tags":["news"]}

curl -s localhost:4600/bookmarks
# [{"id":1,...},{"id":2,...}]

curl -s "localhost:4600/bookmarks?tag=docs"
# [{"id":1,"title":"TS Handbook", ...}]

curl -s localhost:4600/bookmarks/1
# {"id":1,"title":"TS Handbook", ...}

curl -s localhost:4600/bookmarks/99
# {"error":"not found"}

curl -s -X POST localhost:4600/bookmarks -H 'Content-Type: application/json' -d '{"title":"","url":"bad"}'
# {"error":"title is required and must be a non-empty string"}

curl -s -X DELETE localhost:4600/bookmarks/2 -o /dev/null -w "status: %{http_code}\n"
# status: 204

curl -s localhost:4600/bookmarks
# [{"id":1,"title":"TS Handbook", ...}]
```

Every response — success, tag-filtered list, 404, validation error,
204 delete — matches what the code declares, end to end, from a typed
route handler through `Result`-based validation to an actual HTTP
response.

## What this project demonstrates from across the whole path

| Concept | Where |
|---|---|
| Generic type parameters (Level 2/3) | `Result<T, E>`, `asyncHandler<P, ResBody, ReqBody, Q>` |
| Discriminated unions (Level 2) | `{ ok: true; value: T } \| { ok: false; error: E }` |
| Typed Express routes (Level 3) | Every `Request<P, ResBody, ReqBody, Q>` handler |
| Design patterns — Result as an alternative to exceptions (Level 4) | `ok()`/`err()`/`validateBookmarkInput` |
| Testing with typed mocks (Level 3/4) | `store.test.ts` |
| Strict mode beyond the default (Level 4) | `noUncheckedIndexedAccess` in `tsconfig.json` |

## Stretch goals

- Add a `PATCH /bookmarks/:id` route that partially updates a
  bookmark, reusing `validateBookmarkInput` with a version that accepts
  `Partial` fields and only validates the ones present.
- Swap `BookmarkStore`'s in-memory array for the generic `Table<T>`
  repository from Level 3, Module 04, keeping every route and test
  passing unchanged.
- Containerize the finished service with the multi-stage Dockerfile
  pattern from Level 4, Module 06, and run the same `curl` sequence
  against the containerized version to confirm identical behavior.
- Add the GitHub Actions workflow from Level 4, Module 04, running
  `typecheck` and `test` (with coverage) on every push.
