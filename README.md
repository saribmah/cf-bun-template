# cf-bun-template

A Bun monorepo template for Cloudflare Workers + React. Opinionated about
**how** code is organized (namespaces, typed errors, OpenAPI from routes),
unopinionated about **what** you build (no DB locked in, no auth provider
locked in beyond JWT bearer).

## What you get

- **Backend** (`packages/api/`) — Cloudflare Worker on Hono + hono-openapi.
  Auto-generated OpenAPI spec, typed errors with `NamedError`,
  `AsyncLocalStorage`-based per-request `Instance` context.
- **SDK** (`packages/sdk/`) — TypeScript SDK auto-generated from the API
  OpenAPI spec via `@hey-api/openapi-ts`. Publishable to npm; consumed by the
  web frontend.
- **Frontend** (`packages/web/`) — React 19 + Vite + TailwindCSS v4. Talks to
  the API exclusively through the generated SDK. Served as static assets by
  the same Worker in production.
- **Tooling** — `oxlint` (linting), `oxfmt` (formatting), `tsgo` (fast
  typecheck), `husky` + `lint-staged` (pre-commit).
- **CI** — GitHub Actions for lint/format/typecheck/test/build, Cloudflare
  deploy, and npm SDK publishing.
- **Agent recipes** — `.agents/*.md` files that walk an AI agent through
  initializing the template, adding features, wiring storage, regenerating
  the SDK, etc.

## Quick start

### With an AI agent (recommended)

```bash
git clone <this-template> my-app
cd my-app
```

Open the directory in your favourite agentic IDE / CLI (Claude Code, Cursor,
etc.) and say:

> Initialize this template for my project.

The agent will read [`.agents/init.md`](./.agents/init.md), interview you for
a few details (app name, package scope, Cloudflare worker name, prod
domain), substitute placeholders, install deps, and run all the checks.

### Manually

```bash
git clone <this-template> my-app
cd my-app

# Replace tokens. Example values shown — pick your own:
APP_NAME=my-app
PKG_SCOPE='@my-app'
WORKER_NAME=my-app
DESCRIPTION='My new app'

# Use sed (BSD on macOS — drop the empty '' after -i for GNU sed):
find . -type f \
  -not -path './node_modules/*' -not -path './.git/*' -not -path './.agents/*' \
  -exec sed -i '' \
    -e "s|{{APP_NAME}}|$APP_NAME|g" \
    -e "s|@app|$PKG_SCOPE|g" \
    -e "s|{{WORKER_NAME}}|$WORKER_NAME|g" \
    -e "s|{{DESCRIPTION}}|$DESCRIPTION|g" \
    {} +

bun install
bun run --filter '*/api' cf-typegen
bun run --filter '*/sdk' build
bun run lint && bun run ts-check && bun run test
rm .agents/init.md   # only useful pre-init
```

## Architecture

See [AGENTS.md](./AGENTS.md) for the full architectural rulebook (also read
by every AI agent that touches this repo). High-level:

```
packages/
├── api/                    # Cloudflare Worker
│   ├── src/
│   │   ├── app/            # Hono app composition
│   │   ├── server/         # routes (HTTP transport) + error mapping
│   │   ├── middleware/     # auth, error handler
│   │   ├── instance/       # AsyncLocalStorage request context
│   │   ├── utils/          # Context, Log, NamedError
│   │   ├── config/         # typed env accessors
│   │   ├── health/         # health endpoint
│   │   └── example/        # reference feature (delete or rename)
│   ├── scripts/            # generate-openapi
│   └── wrangler.jsonc      # Cloudflare config
└── web/                    # React + Vite frontend (delete if API-only)
    └── src/
```

### Backend pattern (per feature)

Each feature is a self-contained namespace. The example below is the
reference shape — clone it for new features.

```
src/<feature>/
  <feature>.ts           # namespace, Entity, errors, operations
  storage.ts             # entitySelect + EntityRow + toEntity + queries
  __tests__/<feature>.test.ts
src/server/routes/<feature>.ts  # router with describeRoute + validator
```

Layering: `routes → feature → storage`, one-way only. Errors are typed
`NamedError` instances mapped to HTTP statuses at the route boundary.

## Common tasks

| Task                       | Recipe                                                              |
| -------------------------- | ------------------------------------------------------------------- |
| Add a feature              | [`.agents/add-feature.md`](./.agents/add-feature.md)                |
| Add a route                | [`.agents/add-route.md`](./.agents/add-route.md)                    |
| Regenerate SDK after API   | [`.agents/regenerate-sdk.md`](./.agents/regenerate-sdk.md)          |
| Wire Prisma + Postgres     | [`.agents/add-storage-prisma.md`](./.agents/add-storage-prisma.md)  |
| Wire Cloudflare D1         | [`.agents/add-storage-d1.md`](./.agents/add-storage-d1.md)          |
| Add SIWE/Privy/OAuth       | [`.agents/add-auth-provider.md`](./.agents/add-auth-provider.md)    |

## Local development

```bash
# In one terminal:
bun run --filter '*/api' dev      # wrangler dev on :8787

# In another:
bun run --filter '*/web' dev      # vite on :3002 (proxies /api → :8787)
```

The web app's Vite config proxies `/api/*` to the local wrangler dev server,
so `fetch('/api/health')` Just Works in dev.

## Deploy

The repo ships with a `Deploy` GitHub Action (`.github/workflows/deploy.yml`)
that runs on pushes to `main`. To enable it, add these secrets:

- `CLOUDFLARE_API_TOKEN`
- `CLOUDFLARE_ACCOUNT_ID`

Manual deploy from your machine:

```bash
bun run --filter '*/api' deploy
```

## Customizing

- **Add custom domains**: edit the `routes` block in `wrangler.jsonc → env.production`.
- **Add Cloudflare bindings** (KV, R2, D1, Queues, DOs, Workflows): edit
  `wrangler.jsonc` then run `bun run --filter '*/api' cf-typegen` to refresh
  the type definitions.
- **Drop the web frontend**: delete `packages/web/`, remove it from
  `package.json → workspaces`, delete the `assets` block from
  `wrangler.jsonc → env.production`.

## License

MIT (or whatever you want — edit this section after cloning).
