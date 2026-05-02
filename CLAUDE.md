# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
# First-time setup (install deps + generate Prisma client + run migrations)
npm run setup

# Development
npm run dev          # Next.js with Turbopack
npm run dev:daemon   # Background server, logs to logs.txt

# Build & lint
npm run build
npm run lint

# Tests (Vitest + jsdom)
npm test                          # run all tests
npx vitest run src/path/to/file   # run a single test file

# Database
npm run db:reset    # drop and re-run all migrations (destructive)
```

Set `ANTHROPIC_API_KEY` in `.env` to use Claude; without it the app falls back to a `MockLanguageModel` that returns static canned components.

## Architecture

### Request / data flow

1. User types a prompt → `ChatInterface` → `POST /api/chat` (`src/app/api/chat/route.ts`)
2. The route reconstructs a `VirtualFileSystem` from the serialized `files` payload, calls `streamText` (Vercel AI SDK) with two tools: `str_replace_editor` and `file_manager`
3. Streamed tool calls are forwarded to the client via `useAIChat` (AI SDK React hook) → `onToolCall` callback → `FileSystemContext.handleToolCall`, which applies mutations to the in-memory VFS
4. On stream finish, if `projectId` + a valid session exist, the full message history and serialized VFS are persisted to the SQLite `Project` row via Prisma

### Virtual File System

`src/lib/file-system.ts` — `VirtualFileSystem` is an in-memory tree (Map-of-Maps). It is the single source of truth for generated files. It serialises to a plain `Record<string, FileNode>` for JSON transport and Prisma storage.

Two AI tools operate on it:
- `str_replace_editor` (`src/lib/tools/str-replace.ts`) — create / view / str_replace / insert commands
- `file_manager` (`src/lib/tools/file-manager.ts`) — rename / delete commands

### Live preview pipeline

`PreviewFrame` (`src/components/preview/PreviewFrame.tsx`) re-renders whenever `refreshTrigger` increments in `FileSystemContext`.

The preview pipeline (`src/lib/transform/jsx-transformer.ts`):
1. Transforms every `.jsx/.tsx/.ts/.js` file in the VFS with `@babel/standalone` (strips TypeScript, converts JSX to `React.createElement`)
2. Creates blob URLs for each transformed module and builds a native ES import map (resolves `@/` alias, third-party packages via `https://esm.sh/*`, placeholders for missing local imports)
3. Renders a full `srcdoc` HTML document inside a sandboxed `<iframe>` — Tailwind CDN injected, React 19 loaded from esm.sh

The entry point is discovered in this order: `/App.jsx` → `/App.tsx` → `/index.jsx` → `/index.tsx` → `/src/App.jsx` → first `.jsx/.tsx` found.

### Context providers

Both providers wrap the page layout (`src/app/main-content.tsx`):

- **`FileSystemProvider`** — owns the `VirtualFileSystem` instance, selected file, and a `refreshTrigger` counter that drives preview re-renders. Exposes `handleToolCall` which interprets raw AI SDK tool-call payloads and mutates the VFS.
- **`ChatProvider`** — wraps `useAIChat`, serialises the current VFS into every request body, and handles anonymous-work tracking via `sessionStorage` (`src/lib/anon-work-tracker.ts`).

### Auth

JWT-based, cookie-stored (`auth-token`, 7-day expiry). `src/lib/auth.ts` is server-only. Middleware (`src/middleware.ts`) guards `/api/projects` and `/api/filesystem`; `/api/chat` is intentionally unprotected (anonymous sessions are allowed). Projects are only persisted for authenticated users.

### Database

Prisma + SQLite (`prisma/dev.db`). Generated client lives in `src/generated/prisma` (not `node_modules`). Two models: `User` and `Project`. `Project.messages` and `Project.data` are JSON-stringified columns (SQLite has no native JSON type).

### AI provider

`src/lib/provider.ts` exports `getLanguageModel()`:
- With `ANTHROPIC_API_KEY` → returns `anthropic("claude-haiku-4-5")` via `@ai-sdk/anthropic`
- Without key → returns `MockLanguageModel`, a hand-rolled `LanguageModelV1` implementation that streams canned component code through the same tool-call flow
