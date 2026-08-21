# 03 · Building APIs with Express + TS

Express itself is plain JavaScript, but the `@types/express` package
gives every request, response, and middleware a real shape. This module
builds a small typed REST API — routes, request bodies, route params,
status codes, and an error handler — and runs it for real with `curl`.

## Setup

```bash
npm install express
npm install -D typescript @types/express @types/node
```

`@types/express` is what makes `Request` and `Response` type-checked;
without it, `req` and `res` fall back to `any` and TypeScript can't help
you at all.

## A typed in-memory users API

```typescript
import express, { Request, Response, NextFunction } from "express";

interface CreateUserBody {
  name: string;
  email: string;
}

interface User {
  id: number;
  name: string;
  email: string;
}

const app = express();
app.use(express.json());

const users: User[] = [];
let nextId = 1;

app.get("/users", (_req: Request, res: Response) => {
  res.json(users);
});

app.get("/users/:id", (req: Request<{ id: string }>, res: Response) => {
  const id = Number(req.params.id);
  const user = users.find((u) => u.id === id);
  if (!user) {
    res.status(404).json({ error: "not found" });
    return;
  }
  res.json(user);
});

app.post(
  "/users",
  (req: Request<{}, {}, CreateUserBody>, res: Response) => {
    const { name, email } = req.body;
    if (!name || !email) {
      res.status(400).json({ error: "name and email required" });
      return;
    }
    const user: User = { id: nextId++, name, email };
    users.push(user);
    res.status(201).json(user);
  }
);

function errorHandler(
  err: Error,
  _req: Request,
  res: Response,
  _next: NextFunction
): void {
  console.error(err.message);
  res.status(500).json({ error: "internal error" });
}
app.use(errorHandler);

app.listen(4400, () => console.log("listening on 4400"));
```

`Request<{ id: string }>` types `req.params`; `Request<{}, {}, CreateUserBody>`
types `req.body` (the middle type parameter is the response body, which
you can leave as `{}` unless you want to constrain what every handler
sends back).

## Running it against a real server

Compiled and started with `node dist/a.js`, then hit with `curl`:

```bash
curl -s -X POST localhost:4400/users \
  -H 'Content-Type: application/json' \
  -d '{"name":"Ada","email":"ada@example.com"}'
# {"id":1,"name":"Ada","email":"ada@example.com"}

curl -s localhost:4400/users
# [{"id":1,"name":"Ada","email":"ada@example.com"}]

curl -s localhost:4400/users/1
# {"id":1,"name":"Ada","email":"ada@example.com"}

curl -s localhost:4400/users/99
# {"error":"not found"}

curl -s -X POST localhost:4400/users -H 'Content-Type: application/json' -d '{"name":""}'
# {"error":"name and email required"}
```

Every response matches what the handler's types say it should — the
`404` branch returns `{ error: string }`, the success branch returns a
full `User`, exactly as declared.

## Traps

**`req.body` is `any` until you tell Express otherwise.** Without the
generic (`Request<{}, {}, CreateUserBody>`), `req.body.name` type-checks
even if you typo it as `req.body.nmae` — the mistake only surfaces at
runtime as `undefined`.

**Route handlers that don't return still need a `void` return path.**
TypeScript's Express types expect handlers to return `void` (or
`Promise<void>`) — if you write `return res.status(404).json(...)`
instead of `res.status(404).json(...); return;`, older `@types/express`
versions could complain because `res.json()` returns a `Response`, not
`void`. Current versions handle both, but the `return;`-only style shown
above is what keeps mixed early-return handlers unambiguous.

**Forgetting `express.json()` doesn't error — it just silently gives you
an empty `req.body`.** There is no type error for this because
`req.body`'s type comes from the generic you supplied, not from whether
parsing middleware actually ran; the failure only shows up as `undefined`
fields at runtime.

**Async handlers that throw don't reach `errorHandler` automatically**
in Express 4 — you must forward the error yourself
(`.catch(next)` or a wrapping helper); Express 5 fixes this by
automatically forwarding rejected promises.

## Cheat sheet

| Type | Purpose |
|---|---|
| `Request<P>` | Types `req.params` as `P` |
| `Request<P, ResBody, ReqBody>` | Also types `req.body` |
| `Response<ResBody>` | Constrains what `res.json()` accepts |
| `NextFunction` | Type of Express's `next` callback |
| `(err, req, res, next) => void` (4 args) | Signature Express recognizes as an error handler |
| `express.json()` | Middleware that populates `req.body` from JSON |

## Exercise

Add a `PATCH /users/:id` route typed with
`Request<{ id: string }, {}, Partial<Pick<User, "name" | "email">>>` that
updates only the fields present in the body, returns `404` for an
unknown id, and `200` with the updated user otherwise. Start the server
and verify all four cases (partial name update, partial email update,
both, and unknown id) with real `curl` calls.
