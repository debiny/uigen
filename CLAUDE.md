# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
npm run dev          # Start dev server with Turbopack
npm run build        # Production build
npm run lint         # ESLint
npm run test         # Run all Vitest tests
npm run setup        # First-time setup: install deps, Prisma generate + migrate
npm run db:reset     # Wipe and reinitialize SQLite database
npm run dev:daemon   # Start dev server in background, logs to logs.txt
```

Run a single test file:
```bash
npx vitest run src/components/chat/__tests__/ChatInterface.test.tsx
```

> **Note:** `build` and `start` scripts require `NODE_OPTIONS='--require ./node-compat.cjs'` (already included in package.json). This patches out Node.js 25+ `localStorage`/`sessionStorage` globals that break SSR.

## Architecture

**UIGen** is a Next.js 15 (App Router) app that lets users generate React components via AI chat with live preview.

### Key directories

- `src/app/` — Pages and API routes. `api/chat/route.ts` is the streaming AI endpoint.
- `src/components/chat/` — Chat UI (input, message list, markdown renderer)
- `src/components/editor/` — Monaco-based code editor + virtual file tree
- `src/components/preview/` — iframe-based live preview (`PreviewFrame.tsx`)
- `src/lib/` — Core logic: virtual file system, AI provider, auth, contexts
- `src/actions/` — Next.js server actions for auth and project CRUD
- `prisma/` — SQLite schema with `User` and `Project` models

### Core data flow

1. User sends a message → `POST /api/chat` with current virtual FS serialized as JSON
2. Claude Haiku 4.5 streams back text and tool calls (`str_replace_editor`, `file_manager`)
3. Tool calls mutate the in-memory `VirtualFileSystem` (no disk writes)
4. File system changes propagate via `FileSystemContext` → `PreviewFrame` re-renders
5. Babel Standalone transforms JSX in the iframe at runtime
6. If authenticated, the project (messages + file data) is persisted to SQLite

### Virtual file system

`src/lib/file-system.ts` — in-memory tree structure. Serializes to/from JSON for database storage. This is how all generated code is stored during a session.

### AI provider

`src/lib/provider.ts` — wraps `@ai-sdk/anthropic`. Falls back to a `MockLanguageModel` if `ANTHROPIC_API_KEY` is not set (returns static Counter/Form/Card samples). The `.env` file configures this key.

### AI tools

Two tools are registered per chat request and executed server-side against the in-memory `VirtualFileSystem`:
- `str_replace_editor` (`src/lib/tools/str-replace.ts`) — view, create, str_replace, insert operations. `undo_edit` is intentionally unsupported.
- `file_manager` (`src/lib/tools/file-manager.ts`) — file/directory management (move, delete, list).

### Preview pipeline

`src/lib/transform/jsx-transformer.ts` — runs entirely in the browser (not server-side):
1. Babel Standalone transforms JSX/TSX to plain JS
2. CSS imports are stripped and collected separately
3. Each file becomes a Blob URL; an ES module import map is built dynamically
4. Third-party packages are resolved via `https://esm.sh/<package>`; missing local imports get placeholder stub modules
5. The iframe receives the full HTML with the import map injected; Tailwind CSS is loaded from CDN (`cdn.tailwindcss.com`) inside the preview only

### Generation constraints

The AI prompt (`src/lib/prompts/generation.tsx`) enforces:
- Entry point must be `/App.jsx` with a default export
- All local imports must use the `@/` alias (e.g., `@/components/Foo`)
- Style with Tailwind only, no hardcoded styles or HTML files

### State management

Two React contexts wrap the entire app:
- `FileSystemContext` (`src/lib/contexts/file-system-context.tsx`) — virtual FS state
- `ChatContext` (`src/lib/contexts/chat-context.tsx`) — chat messages and streaming state (uses Vercel's `useChat`)

### Authentication

JWT tokens in HTTP-only cookies (7-day expiry, `jose` library, bcrypt hashing). Server actions in `src/actions/index.ts` handle sign-up, sign-in, sign-out. Anonymous users can work without an account; their work is tracked in localStorage (`src/lib/anon-work-tracker.ts`) and can be claimed on sign-up.

### Layout

Split-panel layout: Chat (35%) | Preview or Code editor (65%). Preview shows an iframe; Code shows file tree + Monaco editor in a 30/70 sub-split.

### Testing

Vitest + React Testing Library with JSDOM. Tests live in `__tests__/` subdirectories next to the code they test.
