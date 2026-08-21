# 10 · Project — REST API with Express

This project pulls together everything from Level 3: typed Express
routes, a class-based data layer, custom error types, and a central
error handler — a small but complete Task API with full CRUD, built and
run for real.

## Project layout

```text
task-api/
├── package.json
├── tsconfig.json
└── src/
    └── index.ts
```

## `package.json`

```json
{
  "name": "task-api",
  "version": "1.0.0",
  "private": true,
  "main": "dist/index.js",
  "scripts": {
    "build": "tsc",
    "start": "node dist/index.js",
    "dev": "tsc --watch"
  },
  "dependencies": {
    "express": "^4.19.2"
  },
  "devDependencies": {
    "@types/express": "^4.17.21",
    "@types/node": "^20.11.30",
    "typescript": "^5.9.0"
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
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true
  },
  "include": ["src"]
}
```

## `src/index.ts`

```typescript
import express, { Request, Response, NextFunction } from "express";

interface Task {
  id: number;
  title: string;
  done: boolean;
}

interface CreateTaskBody {
  title: string;
}

interface UpdateTaskBody {
  title?: string;
  done?: boolean;
}

class NotFoundError extends Error {
  constructor(message: string) {
    super(message);
    this.name = "NotFoundError";
  }
}

class ValidationError extends Error {
  constructor(message: string) {
    super(message);
    this.name = "ValidationError";
  }
}

class TaskStore {
  private tasks: Task[] = [];
  private nextId = 1;

  list(): Task[] {
    return [...this.tasks];
  }

  create(title: string): Task {
    const task: Task = { id: this.nextId++, title, done: false };
    this.tasks.push(task);
    return task;
  }

  find(id: number): Task {
    const task = this.tasks.find((t) => t.id === id);
    if (!task) throw new NotFoundError(`task ${id} not found`);
    return task;
  }

  update(id: number, patch: UpdateTaskBody): Task {
    const task = this.find(id);
    if (patch.title !== undefined) task.title = patch.title;
    if (patch.done !== undefined) task.done = patch.done;
    return task;
  }

  remove(id: number): void {
    const index = this.tasks.findIndex((t) => t.id === id);
    if (index === -1) throw new NotFoundError(`task ${id} not found`);
    this.tasks.splice(index, 1);
  }
}

const app = express();
app.use(express.json());
const store = new TaskStore();

function asyncHandler<P, ResBody, ReqBody>(
  fn: (req: Request<P, ResBody, ReqBody>, res: Response<ResBody>) => void
): (req: Request<P, ResBody, ReqBody>, res: Response<ResBody>, next: NextFunction) => void {
  return (req, res, next) => {
    try {
      fn(req, res);
    } catch (err) {
      next(err);
    }
  };
}

app.get("/tasks", (_req: Request, res: Response) => {
  res.json(store.list());
});

app.post(
  "/tasks",
  asyncHandler((req: Request<{}, {}, CreateTaskBody>, res: Response) => {
    const { title } = req.body;
    if (!title || typeof title !== "string") {
      throw new ValidationError("title is required");
    }
    res.status(201).json(store.create(title));
  })
);

app.get(
  "/tasks/:id",
  asyncHandler((req: Request<{ id: string }>, res: Response) => {
    res.json(store.find(Number(req.params.id)));
  })
);

app.patch(
  "/tasks/:id",
  asyncHandler((req: Request<{ id: string }, {}, UpdateTaskBody>, res: Response) => {
    res.json(store.update(Number(req.params.id), req.body));
  })
);

app.delete(
  "/tasks/:id",
  asyncHandler((req: Request<{ id: string }>, res: Response) => {
    store.remove(Number(req.params.id));
    res.status(204).end();
  })
);

app.use(
  (err: Error, _req: Request, res: Response, _next: NextFunction): void => {
    if (err instanceof NotFoundError) {
      res.status(404).json({ error: err.message });
      return;
    }
    if (err instanceof ValidationError) {
      res.status(400).json({ error: err.message });
      return;
    }
    console.error(err);
    res.status(500).json({ error: "internal server error" });
  }
);

const PORT = process.env.PORT ? Number(process.env.PORT) : 4500;
app.listen(PORT, () => console.log(`Task API listening on ${PORT}`));

export { app, TaskStore };
```

Two design choices worth calling out: `asyncHandler<P, ResBody, ReqBody>`
is generic specifically so each route keeps its own params/body types
after wrapping — a non-generic wrapper would collapse every route back
to the untyped default `Request`, silently losing the whole point of
typing `req.params`/`req.body` in the first place. And `NotFoundError`/
`ValidationError` as distinct classes let the single error-handling
middleware use `instanceof` to pick the right status code, instead of
scattering `res.status(...)` calls through every route.

## Running it

```bash
npm install
npm run build
npm start
```

```text
Task API listening on 4500
```

Exercised with real `curl` calls against the running server:

```bash
curl -s -X POST localhost:4500/tasks -H 'Content-Type: application/json' -d '{"title":"Write docs"}'
# {"id":1,"title":"Write docs","done":false}

curl -s -X POST localhost:4500/tasks -H 'Content-Type: application/json' -d '{"title":"Ship it"}'
# {"id":2,"title":"Ship it","done":false}

curl -s localhost:4500/tasks
# [{"id":1,"title":"Write docs","done":false},{"id":2,"title":"Ship it","done":false}]

curl -s -X PATCH localhost:4500/tasks/1 -H 'Content-Type: application/json' -d '{"done":true}'
# {"id":1,"title":"Write docs","done":true}

curl -s localhost:4500/tasks/99
# {"error":"task 99 not found"}

curl -s -X DELETE localhost:4500/tasks/2 -o /dev/null -w "status: %{http_code}\n"
# status: 204

curl -s localhost:4500/tasks
# [{"id":1,"title":"Write docs","done":true}]

curl -s -X POST localhost:4500/tasks -H 'Content-Type: application/json' -d '{}'
# {"error":"title is required"}
```

Every path — success, 404 on an unknown id, 204 on delete, 400 on a
missing `title` — matches exactly what the code declares it should do,
because the types and the error classes stayed in sync through the
whole request lifecycle.

## Traps hit while building this

**A non-generic `asyncHandler` breaks per-route param/body typing.**
The first version of this wrapper took `(req: Request, res: Response) => void`
and every route that used it failed to compile with a `Request<{ id: string }, ...>`
is not assignable to `Request<ParamsDictionary, ...>` error — Express's
own route-matching type expects the *specific* handler shape you
declared, and a wrapper typed too broadly can't satisfy it. Making
`asyncHandler` generic over `P`, `ResBody`, and `ReqBody` and inferring
them from the passed-in function fixed it.

**`res.status(204).end()`, not `res.status(204).json(...)`** — a `204
No Content` response must have no body; sending JSON alongside it is a
protocol violation some HTTP clients reject.

## Stretch goals

- Add pagination to `GET /tasks` via `?limit=&offset=` query params,
  typed with a `Request<{}, {}, {}, { limit?: string; offset?: string }>`
  signature (the fourth generic parameter is query params).
- Add a `PUT /tasks/:id` route that replaces a task wholesale (requires
  both `title` and `done`, unlike `PATCH`'s partial update) and gives it
  its own request-body interface rather than reusing `UpdateTaskBody`.
- Swap the in-memory `TaskStore` for the `Table<T>` generic repository
  from Module 04, and confirm the same routes still pass the same
  `curl` checks with zero route-level changes.
