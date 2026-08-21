# 02 · Framework Integration Patterns

Express, Koa, Fastify, and most server frameworks share a shape: a
chain of middleware functions passed a context and a `next` callback.
This module builds that pattern from scratch, generically over an
app-specific state type — the same technique frameworks use so
`ctx.state` (Koa) or `req.user` (Express, via augmentation) can be
typed per-project rather than hardcoded.

## A generic middleware chain

```typescript
interface Context<State> {
  state: State;
  path: string;
}

type Middleware<State> = (ctx: Context<State>, next: () => void) => void;

class App<State> {
  private middlewares: Middleware<State>[] = [];

  use(mw: Middleware<State>): this {
    this.middlewares.push(mw);
    return this;
  }

  handle(ctx: Context<State>): void {
    let index = -1;
    const run = (i: number): void => {
      if (i <= index) throw new Error("next() called multiple times");
      index = i;
      const mw = this.middlewares[i];
      if (!mw) return;
      mw(ctx, () => run(i + 1));
    };
    run(0);
  }
}
```

`App<State>` doesn't know what `State` is — that's the whole point.
Each project instantiates `App<MyAppState>` and every middleware
registered on it gets `ctx.state` typed as `MyAppState` automatically,
with no casts.

## Using it with a project-specific state

```typescript
interface AppState {
  userId?: string;
  logs: string[];
}

const app = new App<AppState>();

app.use((ctx, next) => {
  ctx.state.logs.push(`request to ${ctx.path}`);
  next();
});

app.use((ctx, next) => {
  ctx.state.userId = "user-42";
  next();
});

app.use((ctx) => {
  ctx.state.logs.push(`handled for ${ctx.state.userId}`);
});

const ctx: Context<AppState> = { state: { logs: [] }, path: "/dashboard" };
app.handle(ctx);
console.log(ctx.state.logs);
console.log(ctx.state.userId);
```

```text
[ 'request to /dashboard', 'handled for user-42' ]
user-42
```

Every middleware's `ctx` parameter is `Context<AppState>` — typing
`ctx.state.usrId` (typo) inside any of these functions is a compile
error, exactly as if `AppState` had been hardcoded, but the same `App`
class is reusable for a completely different `State` shape elsewhere.

## A plugin system built on the same generic

```typescript
interface BasePlugin<State> {
  name: string;
  install(app: App<State>): void;
}

const loggerPlugin: BasePlugin<AppState> = {
  name: "logger",
  install(pluginApp) {
    pluginApp.use((c, next) => {
      console.log(`[App] visiting ${c.path}`);
      next();
    });
  },
};

const app2 = new App<AppState>();
loggerPlugin.install(app2);
app2.use((ctx, next) => {
  ctx.state.logs.push(`handled ${ctx.path}`);
  next();
});
app2.handle({ state: { logs: [] }, path: "/settings" });
```

```text
[App] visiting /settings
```

`BasePlugin<State>` mirrors the app's own generic parameter, so a
plugin written for `AppState` can't accidentally be installed on an
`App<SomeUnrelatedState>` — `loggerPlugin.install(unrelatedApp)` is a
compile error if the state types don't match.

## Traps

**Forgetting to call `next()` silently stops the chain** — there's no
type-level way to enforce that a middleware calls `next()`, since
"handle the response and stop" is a legitimate final step too. In the
example above, the third middleware in the first chain deliberately
doesn't call `next()`; if a fourth middleware had been registered after
it, it would never run, and nothing about the types would warn you.

**Calling `next()` twice re-runs downstream middleware from that point
a second time** unless you guard against it — the `if (i <= index) throw`
check in `run()` exists specifically to catch this common bug (usually
from an `if`/`else` that calls `next()` in both branches by mistake)
loudly instead of silently double-processing a request.

**A generic class's methods returning `this` don't propagate to a
differently-typed variable.** If you do
`const typed: App<AppState> = app.use(...)`, that's fine, but assigning
`app` itself to a variable typed as `App<unknown>` loses the ability to
register more `AppState`-aware middleware on it — the concrete generic
argument only flows one direction.

**Extending a real framework's context type (e.g. Express's `Request`)
uses declaration merging, not generics** — see Module 08 of Level 3.
The generic `App<State>` pattern here is for building your own
framework or an internal service layer; when integrating with an
*existing* framework whose types you don't control, augmentation is
usually the tool, not a generic wrapper class.

## Cheat sheet

| Pattern | Where it shows up |
|---|---|
| `class App<State>` with a `Context<State>` | Koa-style typed app state |
| `type Middleware<State> = (ctx, next) => void` | Any onion-style middleware chain |
| `interface BasePlugin<State>` matching the app's generic | Plugins that stay in sync with app state |
| Guard against double `next()` calls | Prevents silent double-processing bugs |
| Declaration merging (Level 3, Module 08) | Extending an existing framework's types, not building your own |

## Exercise

Add error handling to `App<State>`: wrap the `mw(ctx, ...)` call in
`run()` in a `try`/`catch`, and add an `onError(handler: (err: unknown, ctx: Context<State>) => void)`
method that registers a single error handler invoked when any
middleware throws. Register a middleware that deliberately throws, add
an error handler that pushes the error message into `ctx.state.logs`,
and confirm with a real run that the log array captures it.
