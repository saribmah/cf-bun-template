# Init recipe (for AI agents)

You are about to initialize a fresh clone of `cf-bun-template`. The user said
something like "initialize this template", "set this up for my project", or
"run init". Follow this recipe end-to-end. Be brief and conversational.

## Step 1 — Detect freshness

The template is **uninitialized** if any of these tokens still exist anywhere
in the repo (excluding `node_modules`, `.git`, and `.agents/`):

- `{{APP_NAME}}` — app/workspace name
- `{{WORKER_NAME}}` — Cloudflare worker name
- `{{DESCRIPTION}}` — package description
- `{{APP_DOMAIN}}` — production domain
- `@app/` — placeholder package scope (used in code/imports because curly-brace
  placeholders aren't valid TS module specifiers)

Run:

```bash
grep -rl '{{\|@app/' . --exclude-dir=node_modules --exclude-dir=.git --exclude-dir=.agents
```

If no matches, tell the user the template is already initialized and stop.

## Step 2 — Interview the user

Ask the following questions, one round if you can group them, otherwise one at
a time. Keep tone friendly. Capture answers verbatim.

1. **App name** — short, slug-friendly, lowercase. Example: `mango`.
   Used as the workspace package name and shown in UI titles.
2. **Package scope** — npm-style scope **with** the `@` prefix.
   Example: `@mango`. Workspace packages will be `@mango/api`, `@mango/web`.
3. **One-line description** — for `package.json` and the README.
4. **Cloudflare worker name** — the value you want for `wrangler.jsonc → name`.
   Defaults to the app name. The dev environment becomes `<name>-dev`.
5. **Production domain** (optional) — e.g. `mango.app`. If skipped, leave the
   `routes` block in `wrangler.jsonc` empty so the worker uses
   `<name>.workers.dev` until the user adds a custom domain.
6. **Include the web frontend?** — y/n. Default y. If n: delete `packages/web`,
   remove it from the root `package.json` workspaces, delete the `assets` block
   in `wrangler.jsonc`, drop the proxy from `vite.config.ts` (n/a once removed).
7. **Initial git remote** (optional) — if provided, run `git remote add origin <url>`.

## Step 3 — Substitute placeholders

For each token, replace **every** occurrence across the repo, excluding
`node_modules`, `.git`, and `.agents/`:

| Token             | Source                                            |
| ----------------- | ------------------------------------------------- |
| `{{APP_NAME}}`    | answer 1                                          |
| `@app`            | answer 2 (e.g. `@mango`) — substitute as a string |
| `{{DESCRIPTION}}` | answer 3                                          |
| `{{WORKER_NAME}}` | answer 4                                          |
| `{{APP_DOMAIN}}`  | answer 5 (or leave the routes block commented)    |

**Note on `@app`**: this is a real placeholder string that appears in
package.json `name` fields, TS imports, and `.oxlintrc.json`. Substitute it
with the user's chosen scope (e.g. `@mango`). Be careful to match only the
template's `@app` and not any unrelated `@app...` strings the user may add
later (none exist in the fresh template).

Run substitutions with `find` + `sed` per token, or use the Edit tool
file-by-file. Verify afterwards:

```bash
grep -rl '{{\|@app/' . --exclude-dir=node_modules --exclude-dir=.git --exclude-dir=.agents
```

Should produce no matches.

## Step 4 — Optional: drop web

If the user said no to web (Q6):

```bash
rm -rf packages/web
```

Then edit `package.json` and remove `"packages/web"` from `workspaces.packages`.
In `packages/api/wrangler.jsonc`, delete the `assets` block under
`env.production`. In `.github/workflows/deploy.yml`, no changes needed (it only
deploys the API).

## Step 5 — Capture project details

Write `.agents/PROJECT.md` from the captured answers. Use this template:

```markdown
# Project

- **Name**: <app name>
- **Scope**: <pkg scope>
- **Description**: <description>
- **Worker name**: <worker name>
- **Production domain**: <domain or "not set">
- **Web frontend**: <yes/no>
- **Initialized at**: <ISO date>

Update this file as the project evolves. AI agents read it for context.
```

## Step 6 — Install + verify

The shipped `bun.lock` references the placeholder package names. Delete it
before reinstalling so the lockfile gets regenerated under the real names.

```bash
rm -f bun.lock
bun install
bun run --filter '*/api' cf-typegen   # regenerate worker-configuration.d.ts
bun run --filter '*/sdk' build        # regenerate the SDK from the API
bun run lint
bun run ts-check
bun run test
```

Fix anything that fails before declaring done. Common issues:
- `tsgo` complaining about missing types → run cf-typegen
- `tsgo` failing to resolve `@app/sdk` → confirm Step 3 substituted `@app` to
  the user's real scope in `package.json`s, `sdk.provider.tsx`, and `.oxlintrc.json`
- oxlint complaining about the no-restricted-imports pattern → make sure
  `.oxlintrc.json` has the actual scope, not `@app`

## Step 7 — Initial commit

If the user provided a remote (Q7), wire it up but **do not push**:

```bash
git remote add origin <url>   # only if Q7 answered
git add -A
git commit -m "chore: initialize from cf-bun-template"
```

If the repo wasn't a git repo yet, `git init` first.

## Step 8 — Self-cleanup

Delete this file (`.agents/init.md`) — it's only useful pre-init. Keep the
other `.agents/*.md` files; those are ongoing recipes. Confirm by:

```bash
rm .agents/init.md
```

## Step 9 — Tell the user what's next

Print a short summary:

- ✅ Template initialized as `<app name>` (`<scope>/api`, `<scope>/web`)
- ✅ `bun install` complete, lint/ts-check/test passing
- Next: copy `.env.example` to `.env` (or create one) with `JWT_SECRET=<random>`
- Next: `bun run --filter <scope>/api dev` to start the API on `:8787`
- Next: `bun run --filter <scope>/web dev` to start the web app on `:3002`
- Reference: `AGENTS.md` for canonical patterns; `.agents/add-feature.md` to add a feature
