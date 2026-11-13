# Working agreement for coding agents

Project: <PROJECT TITLE>. Read `docs/SPEC.md` first, then the roadmap file the
maintainer names. Check `docs/TODO.md` for open items flagged **[Agent]** or **[Operator]**.

If the maintainer says "get started", read `docs/roadmap/v1.md` and see where we are.

## Where things live

- `docs/SPEC.md`: the stable contract.
- `docs/TECH_STACK.md`: the shared KIFF stack. Project deltas are in SPEC section 12.
- `docs/roadmap/v1.md`, `docs/roadmap/v2.md`: milestones and checklists.
- `docs/DECISIONS.md`: one entry per non-obvious choice, newest at the bottom.
- `docs/TODO.md`: open work, flagged [Operator] (needs the maintainer) or [Agent].
- `docs/REFERENCES.md`: docs, sources, and upstream issues behind the spec.
- `docs/ISSUES.md`: upstream bugs being tracked or drafted.
- `server/protocol/`: Zod schemas. The source of truth for `docs/PROTOCOL.md`.
- `firmware/`: plain Lua that runs on CC: Tweaked computers and turtles.

## Session rules

- One milestone per session. Do not build ahead.
- Before writing code, restate the plan in under 10 lines and list the VERIFY items
  you will check. Wait for a go-ahead if a decision from SPEC section 15 is involved.
- Read the relevant Void guide page (https://void.cloud/guide/, or the version-matched
  copy in `node_modules/void/docs/`) before using any Void feature. If the docs and
  SPEC disagree, trust the docs and report the difference.

## Style

- Simple and readable beats clever and fast. Code should read like a good tutorial:
  small functions, clear names, comments that explain why. No jokes in comments.
- Queen and colony theming lives in UI copy and docs only. Code uses plain names
  (SPEC section 5).
- TypeScript everywhere on the hub. Plain Lua in `firmware/`, no build step, no
  Lua dependencies.
- When simplicity and correctness conflict, flag the tradeoff in a comment and in your
  reply. Never silently pick either side. No lock-free or clever concurrency.

## Safety

- Never commit credentials. Never log tokens or the turtle shared secret.
- Firmware safety rails (SPEC 8.3) are never relaxed to make a test pass.
- Turtles never get commands that target entities or players.

## End of session

- Update `docs/DECISIONS.md` (one short entry per non-obvious choice), tick the milestone
  checklist, and update `docs/TODO.md`. Summarize what is done, what is blocked, and
  what you recommend next.

## Void

- This project uses [Void](https://void.cloud), a full-stack Vite framework for Cloudflare Workers with file-based API routing, server-rendered pages, and typed backend services.
- Use Void's CLI and typed APIs. Before working with Void, read the relevant Markdown docs in `node_modules/void/docs/`; start with `guide/quickstart.md` for setup and `reference/cli.md` for commands.
