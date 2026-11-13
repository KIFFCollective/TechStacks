# FULL_STACK_CLOUDFLARE

A reusable description of KIFF's standard Full Stack Cloudflare setup.

## 1. How to use this file

- The **core** (sections 2 to 7) applies to every project on this stack.
- The **modules** (section 8) are opt-in. Delete the ones a project does not use.
- A project is either a single Void app (the default) or a multi-package repo of
  libraries and apps (module 8.5). In a multi-package repo, the Void parts of the core
  apply to each app, and library packages follow module 8.6 instead.
- Section 9 lists traps we already hit. Read it before the first session.
- Project-specific facts (networks, data model, MCP tools) stay in that project's
  `docs/SPEC.md`, not here.
- When a project learns something that applies to every project, update the master copy
  of this file first, then pull it into the other projects.

## 2. Principles

1. Simple and readable beats clever and fast. Pick the version a newcomer understands.
2. Lego blocks: prefer what Void and Cloudflare already provide over custom code.
3. Boring dependencies. Ask before adding any. Aim for dependencies that "think" like this stack does.
4. When simplicity and correctness conflict, flag the tradeoff in a comment and in the
   reply. Never silently pick either side.
5. No lock-free tricks. Concurrency control is plain database status transitions
   (conditional `UPDATE ... WHERE status = ?` and check the changed row count).
6. One service layer. Routes, pages, and any other entry points (MCP tools, crons) are
   thin wrappers over `server/services/*`. Business logic lives nowhere else.
7. Never trust written-down external API facts blindly. Anything marked VERIFY is checked
   against current official docs before it is relied on.
8. Cheap by default: aim for the Cloudflare free tier.

## 3. Core stack

| Concern         | Choice                                                       | Minimum Version| Notes                                                                                      |
| --------------- | ------------------------------------------------------------ | -------------- | ------------------------------------------------------------------------------------------ |
| Runtime         | Cloudflare Workers                                           | n/a            | Deployed to the owner's own Cloudflare account                                             |
| App framework   | Void, native app mode (not framework mode)                   | `1.4.0`        | https://void.cloud/guide/ . Version-matched docs in `node_modules/void/docs/`              |
| Package manager | pnpm, for every project                                      | `12.16.1`      | 
| Toolchain       | Vite+ (`vite-plus`, `vp` CLI): Vite, Vitest, Oxlint, Oxfmt   | `1.4.0`        | `vite` is overridden in `pnpm-workspace.yaml` to `npm:@voidzero-dev/vite-plus-core@1.4.0`  |
| Language        | TypeScript 7, strict                                         | `7.0.2`        | Type-checked by `vp check` (`.ts` files only, see 9.4)                                     |
| UI              | Svelte 5 through Void pages (`@void/svelte`, `voidSvelte()`) | `5.57.2`       | Server-rendered loaders and actions first. No React, no UI kits, no client state libraries |
| Server routes   | Void `routes/` (Hono underneath)                             | hono `4.13.13` | Handlers may return a raw `Response`                                                       |
| Validation      | `zod`                                                        | `4.6.7`        | Env, forms, and tool inputs                                                                |
| Database        | D1 + Drizzle via `void/db`                                   | via Void       | Migrations from `void db generate`, committed                                              |
| IDs             | UUID v7                                                      | n/a            | Workers only has `crypto.randomUUID()` (v4). Ask before adding a package                   |
| Secrets         | `env.ts` + `void secret put`                                 | n/a            | Never committed, never logged                                                              |
| Deploy          | `void deploy`                                                | n/a            | Commit `void.lock.json`. Debug deploys with `void project logs`                            |
| Versioning      | Semantic Versioning, first release `0.1.0`                   | n/a            | Until `1.0.0`, minor versions may break things                                             |

All of the above are MIT or Apache-2.0, so they are compatible with AGPL-3.0 projects.
Check each new dependency's license before adding it.

## 4. Repo layout

Void conventions win where they differ. This is the single app layout. A multi-package
repo puts one of these per app under `apps/` (see 8.5).

```
void.config.ts           Worker settings (compatibility date and flags)
vite.config.ts           Vite+ config: plugins, fmt, lint, test
env.ts                   typed env schema
auth.ts                  Better Auth customization (if the auth module is used)
db/schema.ts             Drizzle schema
db/migrations/           generated by `void db generate`, committed
routes/                  API routes (Hono). `.well-known/` is the only dot folder Void reads
pages/                   Svelte pages, `*.svelte` + `*.server.ts`
crons/                   scheduled jobs (if used)
queues/                  queue consumers (if used)
container/               sandbox image: Dockerfile and its own package.json (if used)
server/
  services/              all business logic
  ...                    small single-purpose helpers
client/                  browser-only helpers
scripts/                 manual smoke scripts, never run in CI
test/                    `*.test.ts` unit tests
docs/                    SPEC, DECISIONS, TODO, REFERENCES, ISSUES, TECH_STACK
docs/roadmap/            v1.md, v2.md ...
README.md  AGENTS.md  LICENSE.md
```

## 5. Config files to copy as-is

These are project-independent. Copy them from Postarella and adjust only the marked parts.
Postarella predates the switch to pnpm, so take `pnpm-workspace.yaml` and
`.gitattributes` from this folder or from Bouncer instead.

| File                  | What it does                                                                 | Adjust                                  |
| --------------------- | ---------------------------------------------------------------------------- | --------------------------------------- |
| `.npmrc`              | `save-exact=true` (pnpm reads `.npmrc` too)                                  | nothing                                 |
| `pnpm-workspace.yaml` | overrides, `allowBuilds`, `peerDependencyRules` (see below)                  | `packages` (multi-package repos only)   |
| `.editorconfig`       | UTF-8, LF, two-space indent; Markdown keeps trailing spaces                  | nothing                                 |
| `.gitattributes`      | `* text=auto eol=lf`, binaries marked, lockfile and migration meta generated | nothing                                 |
| `.gitignore`          | ignores `.void/`, `.wrangler/`, `.void-wrangler.jsonc`, `.env*`, `dist/`     | nothing                                 |
| `tsconfig.json`       | extends `.void/tsconfig.json`, strict, `allowImportingTsExtensions`          | `include` list                          |
| `vite.config.ts`      | Void plugins off under Vitest, type-aware lint, fmt ignores                  | `fmt.ignorePatterns`                    |
| `void.config.ts`      | compatibility date and flags                                                 | date; flags only if a module needs them |

Every project uses pnpm, including a single app. Commit `pnpm-lock.yaml`; there is no
`package-lock.json`.

- pnpm minimum version with `"packageManager": "pnpm@12.16.1"` in the root `package.json`. Node 25
  and later have no Corepack, so install pnpm directly
  (`npm install -g pnpm@12.16.1`; this is the only `npm` command a project needs). In
  CI, `pnpm/action-setup` reads `packageManager`.
- Install with `pnpm install`, add dependencies with `pnpm add <name>`, and run scripts
  with `pnpm run <script>` (trap 9.19). CI installs with
  `pnpm install --frozen-lockfile`.
- `pnpm-workspace.yaml` holds the settings npm would keep in `package.json`. A single
  app needs it too, without a `packages` list:
  ```yaml
  overrides:
    vite: npm:@voidzero-dev/vite-plus-core@1.4.0
  allowBuilds: # see trap 9.17
    esbuild: true
    sharp: true
    workerd: true
    better-sqlite3: false
  peerDependencyRules: # the vite override is intended, so do not warn about it
    allowedVersions:
      vite: 1.4.0
  ```
  VERIFY on the first single-app project: these settings are proven in the multi-package
  Bouncer repo (module 8.5), not yet in a single-app repo.

`package.json` scripts (single app; in a multi-package repo these go in each app):

```json
{
  "dev": "vp dev",
  "build": "vp build",
  "preview": "vp preview",
  "check": "vp check",
  "fix": "vp check --fix",
  "test": "vp test run",
  "db:generate": "void db generate",
  "deploy": "void deploy",
  "postinstall": "void prepare"
}
```

`vite.config.ts` shape:

```ts
import { defineConfig } from "vite-plus";
import { voidPlugin } from "void";
import { voidSvelte } from "@void/svelte/plugin";

// Loading the Void plugins under Vitest starts a second dev server and fails.
const isUnitTest = process.env.VITEST === "true";

export default defineConfig({
  plugins: isUnitTest ? [] : [voidPlugin(), voidSvelte()],
  fmt: { ignorePatterns: ["LICENSE.md", "AGENTS.md", "db/migrations/**", "dist/**"] },
  // `vp check` type-checks .ts files only when both options are on.
  lint: { options: { typeAware: true, typeCheck: true } },
  test: { include: ["test/**/*.test.ts"] },
});
```

## 6. Code style

- TypeScript everywhere. Local imports use explicit `.ts` extensions (see 9.3).
- Small functions, clear names, comments that explain why. No jokes in comments.
- Product theming (names, playful copy) lives in UI copy and docs only. Code uses plain
  names. Keep a vocabulary table in the SPEC mapping theme words to code names.
- Wrap any use of an undocumented framework internal in one file with a loud error and
  a comment linking the upstream issue (example: `server/void-auth.ts`).
- Tests import from `vite-plus/test`. Unit tests cover plain functions; anything needing
  the running app goes in a smoke script under `scripts/`.
- Tests never call real third-party services by default. Mock `fetch`.

## 7. Working agreement and docs system

Every project carries the same doc set. The agent reads them in this order.

| File                 | Role                                                                                            |
| -------------------- | ----------------------------------------------------------------------------------------------- |
| `AGENTS.md`          | Session rules for coding agents (copy from Postarella, change the names)                        |
| `docs/SPEC.md`       | Stable contract: goals, non-goals, principles, stack, architecture, open decisions              |
| `docs/roadmap/vN.md` | Milestones with checklists. M0 is a foundation that proves the stack end to end, deployed |
| `docs/DECISIONS.md`  | One dated line per non-obvious choice, newest at the bottom                                     |
| `docs/TODO.md`       | Open work flagged **[Agent]** or **[Operator]**                                                 |
| `docs/REFERENCES.md` | Docs and upstream issues behind the spec                                                        |
| `docs/ISSUES.md`     | Upstream bugs being tracked                                                                     |
| `docs/TECH_STACK.md` | This file                                                                                       |

Upgrade policy: upgrade dependencies deliberately, one at a time where practical, with a
DECISIONS entry listing what was checked (`vp check`, `vp test`, `vp build`, and the dev
requests that matter). Remove workarounds the upgrade makes unnecessary.

## 8. Optional modules

### 8.1 Auth (Better Auth, configured by Void)

This module is for an app that owns its own users. If the app should share logins with
other company apps, use module 8.7 instead and do not add this module.

- `auth.ts` with `defineAuth`. Native Void apps only. Spread `defaults` explicitly; Void
  does not deep merge.
- Plugin tables must appear in `void db generate` output.
- Plugin versions must match the Better Auth version Void bundles (`1.7.18` at
  Void 1.4.0; Void requires `^1.7.11`).
- Closed sign-up (single-owner or invite-only apps): a
  `databaseHooks.user.create.before` hook that throws `APIError` when the new user is
  not allowed (a user already exists, or no invite matches). It covers every sign-up
  path. Test it.
- Do not rely on Better Auth's session cookie cache when other apps check sessions
  against this server; a logged-out user could still pass until the cache expires.
- VERIFY per project: extra plugins (API key, organization, OAuth provider) and
  `advanced.crossSubDomainCookies` work through `defineAuth`. Cross-subdomain cookies
  need a custom domain, because `workers.dev` is on the Public Suffix List.
- Rate limits: keep the defaults, and set `rateLimit: { storage: "database" }` so all
  Worker instances share one count. Void only rate limits deployed requests.
- Routes that need the auth instance itself go through one wrapper around
  `c.get("__voidAuth")` until Void ships a public accessor (void#661).

### 8.2 Remote MCP server

- Stateless route with the official SDK v2 (`@modelcontextprotocol/server` `2.3.1`,
  `createMcpHandler`, default `legacy: "stateless"`). No Durable Objects.
- OAuth: Better Auth `mcp()` plugin plus `jwt()`. Do not also add `oauthProvider()`.
  Add `@better-auth/oauth-provider` as a direct dependency only if pages import its
  client plugin.
- Client registration: open DCR for now. CIMD needs an SSRF-safe fetcher that Workers
  cannot provide yet.
- Void only forwards `/api/auth/*`, so add `routes/.well-known/[...path].ts` to pass the
  discovery URLs through. Issuer is `<origin>/api/auth`.
- Derive the resource URL (token audience) from the request origin so every environment
  works without config.
- Needs `global_fetch_strictly_public` (see 9.1).
- Keep a manual end-to-end script (`scripts/smoke-mcp-oauth.ts`) that walks the whole
  OAuth flow against a running server.

### 8.3 Storage, scheduling, queues

- Files: R2 via `void/storage`. Never pass large files through a Worker request body.
- Scheduling: Void `crons/` (`export const cron = "..."` + `defineScheduled`). Test with
  the local trigger endpoint `/__void/scheduled`.
- Queues: Void `queues/`. Void configures no dead-letter queue. Test with `/__void/queue`.
- Reads that decide what work to do must use the D1 primary
  (`withSession("first-primary")`), because Void turns on read replication.

### 8.4 Sandboxes (containers)

Use when a project must run native binaries or long commands that a Worker cannot.
Source: `node_modules/void/docs/guide/sandboxes.md` and `reference/config.md` (`sandbox`).

- A Void Sandbox is a Cloudflare Container behind a Durable Object. Importing
  `getSandbox` from `void/sandbox` enables it. Do not hand-write the Durable Object or
  the container binding.
- Cost: needs Workers Paid and Containers access. This breaks principle 8, so record it
  in `docs/DECISIONS.md` when a project adopts the module.
- Docker must be running for `vp dev` and for `void deploy` (the deploy builds the
  image). CI that deploys needs Docker too.
- `getSandbox(id, options)` does not start anything; the first command or file operation
  does. IDs are 1 to 63 characters and each ID is its own isolated container. Use one ID
  per unit of work when work must not share state.
- Options: `enableInternet` (default `false`, keep it off unless needed),
  `inactivityTimeoutMs` (default 10 minutes, maximum 6 hours), `labels`.
- Commands are an executable plus an argument array: `sandbox.run(['tool', '--flag', x])`.
  Never wrap user input in `['sh', '-c', ...]`.
- Always await `run()` or `exec()` then `process.output()`. Unattended background
  commands can stop when the sandbox goes idle. Pass `signal: AbortSignal.timeout(ms)`.
- Move files through the Worker: stream R2 objects in with `sandbox.files.writeFile()` and
  stream results out with `(await sandbox.files.readFile(path)).body` into
  `storage.put()`. The sandbox then needs no network and no R2 credentials.
- Container disk, files and processes are lost on idle stop, crash or restart. Save
  anything that matters to D1 or R2 before returning. Call `sandbox.destroy()` in a
  `finally` block when the work is done.
- Limits reject with `SandboxLimitError` (extends `UsageLimit` from `void/limits`).
  Void routes return a structured 429 when it is uncaught. In queue consumers, catch
  it and `msg.retry({ delaySeconds })`.
- Custom image: set `sandbox.image` (a Dockerfile path) and `sandbox.instanceType`
  (`lite`, `standard-1` to `standard-4`; `standard-4` is 4 vCPU, 12 GiB) in
  `void.config.ts`. One `sandbox` block per app, so one size for every sandbox.
- Keep the image in a top-level `container/` folder with its own `package.json`,
  outside the Void app build.
- The image's final stage must copy the helper that matches Void's Sandbox SDK and must
  not run a server. At Void 1.4.0:
  ```dockerfile
  FROM node:26.12.1-trixie-slim
  COPY --from=docker.io/cloudflare/sandbox:1.0.2 /usr/local/bin/sandbox-shim /usr/local/bin/sandbox-shim
  WORKDIR /workspace
  CMD ["sleep", "infinity"]
  ```
  The reference copy is `node_modules/void/sandbox.Dockerfile`. See trap 9.8.
- Commit `void.lock.json`; the sandbox adds a Durable Object migration. Never reorder
  deployed migration steps.
- VERIFY per project: Cloudflare Containers limits on the owner's account (concurrent
  instances, disk per instance type) and cold start time for the project's image.

### 8.5 Multi-package repos (pnpm workspaces)

Use when one repo holds several packages, such as shared libraries plus the apps that
use them. The pnpm setup from section 5 applies; this module adds workspaces.

Proven in the Bouncer repo on 2026-11-07 with pnpm `12.16.1`, Void `1.4.0`, Vite+
`1.4.0`, on Windows. Copy its root files as the starting point.

- Layout:
  ```
  pnpm-workspace.yaml    packages, overrides, allowBuilds, peerDependencyRules
  package.json           "private": true, "packageManager", root scripts that fan out
  tsconfig.base.json     strict settings every workspace extends
  .npmrc                 save-exact=true
  packages/<name>/       libraries (module 8.6)
  apps/<name>/           Void apps, each with the section 4 layout
  ```
- Every workspace has its own `vite.config.ts` and its own `check`, `fix`, `test` and
  `build` scripts. Each workspace lists `vite-plus` and `typescript` as dev
  dependencies so the `vp` binary is linked there.
- Internal dependencies use `workspace:*`. pnpm replaces it with the real version when
  packing or publishing.
- Root scripts fan out: `pnpm -r run check`, `pnpm -r run test`, `pnpm -r run build`.
  pnpm runs them in dependency order. Run one workspace with
  `pnpm --filter <name> <script>`.
- `pnpm-workspace.yaml` adds the workspace list to the section 5 settings:
  ```yaml
  packages:
    - packages/*
    - apps/*
  ```
- App `tsconfig.json`: `"extends": ["../../tsconfig.base.json", "./.void/tsconfig.json"]`.
  The generated Void tsconfig only adds route types, so the strict settings must come
  from the base.
- Each app keeps its own `void.config.ts`, `vite.config.ts`, `tsconfig.json` and
  `void.lock.json`, and deploys from its own folder with `pnpm run deploy` (trap 9.19).
- Void writes a `.gitignore` into each app on first run. Delete it; the root one covers it.
- Two Void apps running at once need different dev ports and Worker inspector ports
  (trap 9.20).
- Test CI changes from a fresh clone first (trap 9.25).

### 8.6 Library packages

Use for TypeScript packages that are imported by apps rather than deployed.

- No Void, no Svelte. Plain functions over web standard APIs (`Request`, `Response`,
  `fetch`, `crypto`), so a library works in any Worker, page loader or route.
- Same language, lint, format and test rules as the core: TypeScript 7 strict,
  `vp check`, `vp test`, tests in `test/` importing from `vite-plus/test`.
- Runtime dependencies are even more restricted than in apps, because every app bundles
  them. Prefer none. A framework the library integrates with (for example `hono`) is an
  optional `peerDependency`, never a direct one.
- Inside the workspace, `exports` points at the `.ts` source so apps and tests need no
  build step. For publishing, `publishConfig` swaps `exports` to built JavaScript and
  `.d.ts` files in `dist/` (pnpm applies `publishConfig` on pack and publish). Public
  scoped packages also need `access: "public"`. KIFF's npm scope is `@kiffcollective`.
  Publish from CI only.
- Build with `vp pack`, configured in `vite.config.ts` as
  `pack: { entry: ["src/index.ts"], dts: true }`. It bundles to `dist/index.mjs` and
  `dist/index.d.mts`, so explicit `.ts` imports disappear, and workspace dependencies
  stay external. Check what will ship with `pnpm pack` before the first publish. pnpm
  adds the root `LICENSE.md` to the tarball.
- Library `tsconfig.json` extends `../../tsconfig.base.json` and includes `src`, `test`
  and `vite.config.ts`.
- Libraries version in lockstep with each other unless a project decides otherwise.

### 8.7 Shared company auth (Bouncer kit)

Planned, not yet released. Use when the app should share one login, one set of API keys
and one role vocabulary with the company's other apps. Source and contract: the Bouncer
repo's `docs/SPEC.md`.

- The app does not add module 8.1. It has no users table, no Better Auth, and no
  password handling. The shared auth server owns all of that.
- Install `@kiffcollective/bouncer-kit` and `@kiffcollective/bouncer-core` (MIT).
  Create one kit instance in a small `server/` helper with the auth server URL, issuer
  and this app's audience.
- Pages: call `verify(request)` in loaders and actions. Routes: use
  `requirePermission(resource, action)`.
- CSRF for cookie-authenticated writes is handled inside `verify()` by an Origin check.
  Do not add a separate CSRF mechanism; list any extra allowed origins in the kit
  config.
- An MCP server (module 8.2) on such an app is an OAuth resource server only. It accepts
  Bearer tokens issued by the shared auth server through `kit` and does not run its own
  OAuth provider.
- The app must be served on a subdomain of the shared cookie domain to receive the
  session cookie.

## 9. Known traps

1. **Worker fetching its own URL returns 404.** Cloudflare routes a Worker's fetch to its
   own workers.dev hostname to a non-existent origin. Add
   `compatibility_flags: ["global_fetch_strictly_public"]` in `void.config.ts`.
2. **CRLF on Windows breaks `vp check`.** Git for Windows defaults to
   `core.autocrlf=true`. The `.gitattributes` and `.editorconfig` in section 5 fix it.
3. **`void db generate` loads `auth.ts` with Node**, so local imports need explicit `.ts`
   extensions and `allowImportingTsExtensions`.
4. **`svelte-check` does not support TypeScript 7.** `.svelte` files are not type-checked.
   Keep logic in `.ts` files so it is checked; revisit when `svelte-check` catches up.
5. **Vitest fails with the Void plugins loaded.** Skip them when `VITEST === "true"`.
6. **Undocumented internals** (`__voidAuth`) can change on any Void upgrade. Recheck on
   every upgrade.
7. **Deploy problems:** read `void project logs` before guessing. It is a live tail
   only, so store anything needed later (job errors, command stderr) in D1 or R2.
8. **Sandbox shim version drift.** A custom sandbox image must copy the `sandbox-shim`
   that matches Void's Sandbox SDK. A Void upgrade can bump that version. Compare the
   project Dockerfile with `node_modules/void/sandbox.Dockerfile` on every upgrade.
9. **Void queues have no dead-letter queue.** Cloudflare deletes messages that run out
   of retries. Keep job state in D1 and sweep stuck jobs with a cron, or add a
   dead-letter queue under `cloudflare` in `void.config.ts`.
10. **D1 free tier limits are hard limits** (since 2026-09-01). Over 100,000 row writes
    or 5 million row reads in a day, queries fail until midnight UTC instead of being
    billed. Budget writes on Free, or use Workers Paid.
11. **Void room per-connection state is `serializeAttachment()`.** `ctx.connection.setState()`
    survives hibernation, is lost when the socket closes, and is limited to 16 KB.
    Source: `node_modules/void/dist/runtime/ws-server.mjs` (Void 1.4.0).
12. **No public way to reach a Void room from server code.** The runtime uses an internal
    binding name (`void info --json` shows it). Wrap any use in one file with a loud error.
14. **Run the first `void deploy` and every `void secret` command in a separate terminal,
    never through Claude Code's `!` mode.** These commands ask yes/no questions that `!`
    cannot answer. An interrupted `void secret sync` can leave an empty placeholder
    Worker with no secrets, and the first `void deploy` then fails with "Could not verify
    existing Cloudflare Worker". Fix: delete the placeholder Worker (dashboard, or
    `wrangler delete --name <worker> --config .void-wrangler.jsonc`), set the secrets
    again in a real terminal, check them with `void secret list`, then deploy. Claude
    Code's own permission rules also block it from writing secrets, which is correct.
15. **Void infers bindings only from scanned folders** (`routes/`, `crons/`, `src/`,
    `pages/`, `queues/`, `middleware/`, `email/`), not `server/`. A `void/ai` import that
    only lives in `server/` gives "ai: Not available" at runtime. Fix: name the binding in
    `void.config.ts` with `inference: { bindings: { ai: true } }`. (Found in Spike T,
    2026-11-08.) Related: a second `pnpm run dev` moves to port 5174 when an old dev
    server still holds 5173, so check the port before sending test requests (see 9.20).
16. **`void project logs` and `wrangler tail` need the account when the login sees more
    than one.** `void project logs` is a live tail only (`--range` is ignored for direct
    Cloudflare deploys), and with two accounts it can stream nothing without an error.
    `wrangler tail` fails with "More than one account available". Fix:
    `CLOUDFLARE_ACCOUNT_ID=<id> npx wrangler tail <worker> --format pretty`, which shows
    cron runs with their `console.log` lines. (Found in Spike T, 2026-11-08.)
17. **pnpm blocks dependency build scripts.** Install scripts of dependencies do not run
    unless listed under `allowBuilds` in `pnpm-workspace.yaml` (pnpm 12). Void needs
    `esbuild`, `sharp` and `workerd` allowed and `better-sqlite3` set to `false`. The
    workspaces' own `postinstall` (`void prepare`) always runs. pnpm 12 also reports
    "Agent skills awaiting approval: void"; that is optional and unrelated to builds.
18. **pnpm's strict `node_modules`.** A package can only import what it declares. A tool
    that relies on undeclared (hoisted) packages fails under pnpm and works under npm.
    Fix it by declaring the dependency, not by turning on hoisting. Void 1.4.0 and
    Vite+ 1.4.0 had no such problems.
19. **`pnpm deploy` is a pnpm command, not your script.** It copies a workspace package
    somewhere else. Run the app's script with `pnpm run deploy` (or `void deploy` inside
    the app folder).
20. **Two Void dev servers fight over ports.** Both default to dev port 5173 and Worker
    inspector port 9229, then race for 9230, and one crashes with `EADDRINUSE`. Give
    each app a fixed `server.port` with `strictPort: true`, and set
    `process.env.VOID_INSPECTOR_PORT ??= "<port>"` at the top of its `vite.config.ts`.
    That variable is undocumented in Void 1.4.0, so recheck on upgrades. Port 5173 is
    also a common default for other tools (for example a WSL relay), so pick other ports.
21. **A failed Void page action answers 200, not 4xx.** A `ValidationError` re-renders the
    page with status 200 and puts the field errors in the `__VOID_PAGE_DATA__` JSON
    (`errors`), even for a JSON request. Smoke scripts must read the errors, not the
    status (Void 1.4.0).
22. **Better Auth invitations need a verified email.** `listUserInvitations` refuses any
    session whose `emailVerified` is false (`EMAIL_VERIFICATION_REQUIRED_FOR_INVITATION`),
    with no option to turn it off (Better Auth 1.7.18). Without email sending, nothing
    marks users verified unless the server does it (Bouncer DECISIONS, 2026-11-08).
23. **Void dev reads only `.env`.** Variables from the shell's environment never reach the
    dev Worker, by design (Void 1.4.0, `src/shared/dotenv.ts`). To try other values
    locally, edit `.env` and restart `vp dev`.
24. **`void secret put` can block the next deploy.** Each secret change creates a live
    Worker version without Void's metadata. If `void.config.ts` leaves `workers_dev`,
    `preview_urls` or `routes` unset, Void wants to copy them from the live version,
    cannot, and stops with "dashboard-managed routes or visibility" before uploading
    (Void 1.4.0). State all three under `cloudflare:` in `void.config.ts`.
25. **Check CI changes from a fresh clone.** Ignored folders such as `.void/` hide
    problems that only show up on a clean checkout (Bouncer `docs/ISSUES.md`). Before
    pushing a change to install or build, clone the repo to a temporary folder and run
    `pnpm install --frozen-lockfile`, `pnpm check`, `pnpm test` and `pnpm build`.
