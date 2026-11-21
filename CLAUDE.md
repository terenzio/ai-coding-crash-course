# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

Exercise repo for Matt Pocock's AI Coding Crash Course. It holds two unrelated things:

- **Cadence**, a mini-Udemy course platform: React Router 8 (SSR), TypeScript, SQLite via Drizzle, Tailwind 4 + shadcn/ui. Lives in `app/`. See `Architecture.md` for the full request-flow walkthrough.
- **`request-logger/`**, a dependency-free proxy that sits between a coding agent and a model provider's API and writes each request as readable Markdown. Run separately from the app. See `request-logger/README.md`.

## Commands

```bash
npm install
npm run db:migrate && npm run db:seed   # first-time setup; creates ./data.db (gitignored)
npm run dev                             # http://localhost:5173
npm run build
npm run typecheck                       # runs `react-router typegen` first, then tsc
npm test                                # vitest run (globals enabled)
npm run test:watch
npx vitest run request-logger/proxy.test.ts            # single file
npx vitest run -t "name of test"                       # single test by name
npm run db:generate                     # after editing app/db/schema.ts; emits SQL into drizzle/
npm run request-logger                  # start the logging proxy (interactive wizard on first run)
```

`npm run reset <slug>`, `cherry-pick <slug>` and `pull` are course tooling (`ai-hero-cli`) that move the repo between lesson checkpoints on the `live-run-through` branch. Don't run them unprompted: `reset` rewrites the working tree.

`npm run typecheck` is the only type check; there is no lint script. Prettier is configured (`.prettierrc`).

## App architecture (`app/`)

Strict layering, each layer talks only to the one below: **route → (Zod validation) → service → db client → SQLite**.

- **Routes** (`app/routes/*.tsx`, registered in `app/routes.ts`) export `loader`/`action`. They are the only place that touches `request`, `formData`, redirects or status codes. Each route defines its own Zod schema beside its `action` and validates with `parseFormData` from `app/lib/validation.ts`. The session (`app/lib/session.ts`) is read in loaders/actions, never in services.
- **Services** (`app/services/*.ts`) are plain functions holding business logic and queries. They never import route or UI code.
- **DB** (`app/db/index.ts`, schema in `app/db/schema.ts`). The connection is opened lazily and `db` is a `Proxy`, deliberately. Opening `better-sqlite3` at module scope makes the module side-effectful, so Rollup can no longer drop it from the client bundle when a route imports a service only for its loader, and `better-sqlite3` ends up in the browser. Keep `app/db/index.ts` side-effect free.

Route files are flat dot-named (e.g. `instructor.$courseId.modules.$moduleId.tsx`) and several are 1000+ lines.

## request-logger architecture

Files in `request-logger/` are tightly coupled; a change to one usually needs the others:

- `agents.ts`: catalogue of supported coding agents and providers, plus `resolveChoice` / `resolveCustomTarget`, which turn a wizard answer into a proxy target (upstream, setup files, the command to print). Some agents (OpenCode, Pi) have custom-route behaviour in code branches keyed on agent id, not in catalogue data, so "add an agent" is not always one entry.
- `proxy.ts`: the HTTP proxy. Forwards requests, redacts secrets, and logs to Markdown; throttles identical error bursts.
- `render.ts`: converts a captured request/response to Markdown per wire format (`RendererId`: OpenAI, Anthropic, Gemini, raw, …). Whether a request is worth logging (`shouldLogRequest`) lives in `agents.ts`, not here.
- `config.ts` / `config-wizard.test.ts`: the clack-based wizard and the saved choice. `model-discovery.ts` queries a custom upstream for model ids.

Upstream URL handling (path-prefix and trailing-slash normalisation) is implemented separately in `agents.ts`, `proxy.ts` and `model-discovery.ts`, so a fix to one must be checked against the others.

## Notes

- `README.md` says React Router v7; `package.json` and `Architecture.md` are on v8. Trust the latter.
- `.sandcastle/` is `@ai-hero/sandcastle` agent-sandbox tooling (Dockerfile + scripts), not part of the app.
- Mermaid diagram requests are governed by `.github/instructions/mermaid.instructions.md`.

## Agent skills

### Issue tracker

Issues live in GitHub Issues for `terenzio/ai-coding-crash-course` (via the `gh` CLI). See `docs/agents/issue-tracker.md`.

### Triage labels

Default five-label vocabulary (`needs-triage`, `needs-info`, `ready-for-agent`, `ready-for-human`, `wontfix`). See `docs/agents/triage-labels.md`.

### Domain docs

Single-context: one `GLOSSARY.md` + `docs/adr/` at the repo root. See `docs/agents/domain.md`.
