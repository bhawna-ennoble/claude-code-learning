# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

UIGen is a Next.js 15 / React 19 app that lets a user describe a React component in chat and watch it appear in a live preview. The model writes files into a **virtual, in-memory file system** (nothing is written to disk); the preview iframe transpiles those files in-browser with `@babel/standalone`.

## Commands

- `npm run setup` — install deps, generate the Prisma client, run migrations. Run this once after cloning.
- `npm run dev` — start the dev server (Turbopack) at http://localhost:3000.
- `npm run dev:daemon` — same, but backgrounded with output redirected to `logs.txt` (POSIX shell only — the `&` syntax does not work in PowerShell; use `npm run dev` in a separate terminal on Windows).
- `npm run build` / `npm run start` — production build / serve.
- `npm run lint` — `next lint`.
- `npm test` — run Vitest (jsdom). Run a single test file with `npx vitest run path/to/file.test.tsx`, or a single test by name with `npx vitest run -t "name fragment"`.
- `npm run db:reset` — wipe and re-migrate the SQLite database.

The `dev`, `build`, and `start` scripts all set `NODE_OPTIONS=--require=./node-compat.cjs` — this shim deletes `globalThis.localStorage`/`sessionStorage` before SSR. Node 25's experimental Web Storage API exposes those globals server-side in a non-functional state, which trips browser-detection guards in dependencies. Don't remove the require unless you've verified the project is on a Node version that doesn't ship the experimental Web Storage globals.

## Anthropic API key

The `.env` `ANTHROPIC_API_KEY` is **optional**. `getLanguageModel()` in `src/lib/provider.ts` falls back to `MockLanguageModel`, which emits a hardcoded multi-step script (counter / form / card depending on prompt keywords) and is gated to `maxSteps: 4` in the chat route. When debugging behavior that involves real tool-use loops, set the key — the mock will not exercise the same code paths.

The active model is hardcoded at the top of `src/lib/provider.ts` (`const MODEL = "claude-haiku-4-5"`).

## Architecture

### The virtual file system is the central data structure

`src/lib/file-system.ts` defines `VirtualFileSystem`, a `Map<string, FileNode>` with create/read/update/delete/rename plus serialize/deserialize. Three things consume the same instance:

1. **The AI tools** (`src/lib/tools/str-replace.ts`, `src/lib/tools/file-manager.ts`) — server-side, invoked by `streamText` in `src/app/api/chat/route.ts`. They mutate the FS that lives for the duration of one request.
2. **The React context** (`src/lib/contexts/file-system-context.tsx`) — client-side. Its `handleToolCall` mirrors every tool call from the streamed chat response into a *separate* client-side `VirtualFileSystem` instance so the editor and preview update live as the model writes files.
3. **Persistence** (`prisma/schema.prisma` `Project.data`) — the FS is serialized via `serialize()` to a JSON string column and rehydrated with `deserializeFromNodes()` on load.

**Implication:** the server-side and client-side FS instances are independent. The client mirrors mutations by parsing tool calls in the stream — if you add a new tool that modifies files, you must also handle it in `handleToolCall` or the preview will go stale.

### Request flow

`POST /api/chat` (`src/app/api/chat/route.ts`):
1. Receives `{ messages, files, projectId }`. Files are deserialized into a fresh `VirtualFileSystem`.
2. Prepends `generationPrompt` (in `src/lib/prompts/generation.tsx`) as the system message with `cacheControl: ephemeral`.
3. Streams via Vercel AI SDK with two tools wired to the FS instance.
4. In `onFinish`, if a `projectId` and authenticated session exist, persists `messages` and the serialized FS back to `Project`.

The system prompt enforces the project conventions the preview depends on: every project must have `/App.jsx` as the default-export entry point, all non-library imports use the `@/` alias, styling is Tailwind, no HTML files.

### Preview pipeline

`src/components/preview/PreviewFrame.tsx` + `src/lib/transform/jsx-transformer.ts`:
- Reads all files from the client FS.
- Picks an entry point (`/App.jsx` first, then `/App.tsx`, `/index.jsx`, `/index.tsx`, `/src/App.*`).
- Builds an import map and uses `@babel/standalone` (`react` + `typescript` presets) to transpile each file to a blob URL.
- Injects the result into an iframe along with Tailwind via CDN.

The `@/` alias used by generated code is resolved by the import-map step, not by Next/Webpack — generated code never goes through the host app's bundler.

### Auth & projects

- `src/lib/auth.ts` issues JWTs (`jose`) in an `httpOnly` cookie. `JWT_SECRET` falls back to `"development-secret-key"` if unset — set a real secret for any non-local deployment.
- `src/middleware.ts` only gates `/api/projects` and `/api/filesystem` (note: those routes don't currently exist; the middleware is a placeholder).
- Anonymous users: `src/lib/anon-work-tracker.ts` stashes messages + FS data in `sessionStorage`. After sign-up, that data is migrated into a real `Project` row. `src/app/page.tsx` redirects authenticated users to their most recent project (or creates one).
- Server actions live in `src/actions/` (`create-project`, `get-project`, `get-projects`, plus `getUser` re-exported from the index).

### shadcn/ui

`components.json` configures shadcn (`new-york` style, `neutral` base, `lucide` icons). UI primitives are in `src/components/ui/`. Use `cn()` from `src/lib/utils.ts` for class merging.

### Prisma

Generated client output is **non-default**: `output = "../src/generated/prisma"` (see `prisma/schema.prisma`). Import the client via `src/lib/prisma.ts`, not directly from `@prisma/client`. After schema changes run `npx prisma generate` (or `npm run setup`).

## Tests

Vitest with `jsdom`. Tests are colocated under `__tests__/` next to the code they cover (e.g. `src/lib/__tests__/file-system.test.ts`, `src/lib/contexts/__tests__/`, `src/components/chat/__tests__/`). React Testing Library is set up. There is no global setup file — if you need one, add it via `vitest.config.mts` `test.setupFiles`.
