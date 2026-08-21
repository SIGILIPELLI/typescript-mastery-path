# 06 · Deployment with Docker

Shipping a TypeScript service means shipping compiled JavaScript, not
source — and a naive Dockerfile that copies everything and runs
`npm install` in production ends up bloated with dev dependencies,
the TypeScript compiler itself, and source files nobody needs at
runtime. This module builds a multi-stage Dockerfile around the Task
API project from Level 3, Module 10.

## The project being containerized

Reusing `task-api/` exactly as built in Level 3:

```text
task-api/
├── package.json
├── tsconfig.json
└── src/index.ts
```

Verified locally (Level 3, Module 10) that `npm run build && npm start`
produces a server on port 4500 responding correctly to the full CRUD
`curl` suite — that's the artifact this Dockerfile needs to reproduce
inside a container.

## A naive (bad) Dockerfile

```dockerfile
FROM node:20
WORKDIR /app
COPY . .
RUN npm install
RUN npm run build
CMD ["npm", "start"]
```

This works, but ships the entire `node_modules` including
`typescript`, `@types/*`, and every dev tool, plus the full `src/`
tree, inside the final image — often hundreds of megabytes of things
that provide zero runtime value and increase attack surface.

## Multi-stage build

```dockerfile title="Dockerfile"
# --- Stage 1: build ---
FROM node:20-slim AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY tsconfig.json ./
COPY src ./src
RUN npm run build

# --- Stage 2: production ---
FROM node:20-slim AS production
WORKDIR /app
ENV NODE_ENV=production
COPY package*.json ./
RUN npm ci --omit=dev
COPY --from=build /app/dist ./dist

EXPOSE 4500
USER node
CMD ["node", "dist/index.js"]
```

The `build` stage has TypeScript, `@types/express`, and the full source
— everything needed to run `tsc`. The `production` stage starts fresh
from a clean `node:20-slim`, installs **only** production dependencies
(`npm ci --omit=dev` skips `typescript` and `@types/*` entirely), and
copies across just the compiled `dist/` output via
`COPY --from=build`. The final image contains no TypeScript compiler,
no `.ts` source, and no dev tooling — only what `node dist/index.js`
actually needs.

## `.dockerignore`

```text title=".dockerignore"
node_modules
dist
*.log
.git
.env
```

Excluding `node_modules` and `dist` from the build context matters even
though the Dockerfile only `COPY`s specific paths — without a
`.dockerignore`, Docker still uploads the entire directory (including a
possibly stale local `node_modules` or `dist`) to the build daemon
before the `COPY` instructions even run, slowing every build and
risking a stale local build accidentally getting baked in via a
misconfigured `COPY .` elsewhere in the file.

## Health check and non-root user

```dockerfile
HEALTHCHECK --interval=30s --timeout=3s \
  CMD node -e "require('http').get('http://localhost:4500/tasks', r => process.exit(r.statusCode === 200 ? 0 : 1))"
```

`USER node` (already in the Dockerfile above) switches off `root`
before the final `CMD` runs — the official `node` images ship a
pre-created `node` user specifically for this. Running a containerized
Node process as `root` is unnecessary privilege: if the app is
compromised, the attacker inherits whatever the process user can do,
and inside most container runtimes `root` in the container often maps
close to `root` capabilities on the host unless user namespaces are
configured.

## Environment configuration

```dockerfile
ENV PORT=4500
```

```typescript
const PORT = process.env.PORT ? Number(process.env.PORT) : 4500;
```

The app already reads `PORT` from the environment (Level 3, Module 10)
— this is exactly why: a containerized service should take its
listening port from an env var, since the orchestrator (Docker Compose,
Kubernetes, ECS) decides port mapping, not the app itself.

## Traps

**`COPY . .` before `RUN npm ci` invalidates Docker's layer cache on
every source change**, forcing a full dependency reinstall even when
`package.json` didn't change. Copying `package*.json` first, installing,
*then* copying source (as the build stage above does) means Docker
reuses the cached `npm ci` layer whenever only source files changed —
a large real-world build-time difference.

**`npm install` instead of `npm ci` in a Docker build can silently
produce a different dependency tree than what was tested locally** if
`package-lock.json` and `package.json` have drifted — `npm ci` fails
loudly on drift instead, which is what you want to catch before an
image ships.

**Forgetting `--omit=dev` in the production stage's install** still
pulls in `typescript` and `@types/*` even though the final `CMD` never
touches them — the whole point of splitting into stages is defeated if
both stages install the same full dependency set.

**A `HEALTHCHECK` hitting a stateful endpoint (like the task list,
which starts empty and changes) can pass even when a route the health
check doesn't exercise is broken.** A dedicated `/health` route that
does nothing but return `200` is more common in real services than
reusing a business endpoint for this purpose.

## Cheat sheet

| Dockerfile technique | Why it matters for TypeScript specifically |
|---|---|
| Multi-stage build | Compiler and dev deps stay out of the shipped image |
| `npm ci --omit=dev` in the final stage | Excludes `typescript`, `@types/*` from production |
| `COPY --from=build /app/dist ./dist` | Ships compiled JS, never `.ts` source |
| `package*.json` copied before source | Preserves the `npm ci` layer cache across source-only changes |
| `.dockerignore` with `node_modules`, `dist` | Prevents stale local artifacts leaking into the build context |
| `USER node` | Runs the container process unprivileged |

## Exercise

Build the two-stage Dockerfile above locally (`docker build -t task-api .`),
run it with `docker run -p 4500:4500 task-api`, and repeat the full
`curl` CRUD sequence from Level 3, Module 10 against the containerized
server. Then run `docker images` and compare the image size against a
single-stage build using the "naive" Dockerfile from earlier in this
module, to see the multi-stage size difference for yourself.
