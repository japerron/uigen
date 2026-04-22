# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
npm run setup        # First-time setup: install deps, generate Prisma client, run migrations
npm run dev          # Start Next.js dev server with Turbopack
npm run build        # Production build
npm run lint         # ESLint
npm run test         # Run all tests with Vitest
npm run db:reset     # Reset the SQLite database (destructive)
```

To run a single test file:
```bash
npx vitest run src/path/to/file.test.ts
```

The `.env` file only needs `ANTHROPIC_API_KEY`; the app falls back to a mock AI provider if the key is absent.

## Architecture

UIGen is a Next.js 15 (App Router) application that generates React components via Claude AI, with live in-browser preview.

### Request flow

1. User types a prompt in the chat UI → `POST /api/chat` (`src/app/api/chat/route.ts`)
2. The route calls Claude via Vercel AI SDK `streamText()` with two tools:
   - `str_replace_editor` — create/view/edit files in the virtual FS
   - `file_manager` — rename/delete files
3. Tool calls update the **virtual file system** (in-memory, no disk I/O)
4. The preview iframe compiles JSX with Babel Standalone and renders it live
5. On save, the entire FS + messages are serialized to JSON in the SQLite `Project` table

### Key modules

| Path | Role |
|---|---|
| `src/app/api/chat/route.ts` | Streaming AI endpoint; orchestrates tool calls and DB persistence |
| `src/lib/provider.ts` | Selects Claude vs. mock AI model; wraps `@ai-sdk/anthropic` |
| `src/lib/file-system.ts` | Virtual FS — all file operations live here; serializable to/from JSON |
| `src/lib/tools/` | `str_replace_editor` and `file_manager` tool implementations |
| `src/lib/transform/` | Babel-based JSX→JS transformer used by the preview iframe |
| `src/lib/prompts/` | System prompt passed to Claude (uses Anthropic prompt caching) |
| `src/lib/contexts/` | `FileSystemContext` and `ChatContext` (wraps Vercel AI SDK `useChat`) |
| `src/components/preview/PreviewFrame.tsx` | Iframe renderer; builds an import map for module resolution |
| `src/actions/` | Next.js server actions for auth and project CRUD |
| `src/middleware.ts` | JWT-based route protection |

### State management

- **FileSystemContext** owns the virtual FS state and exposes file operation helpers to all components.
- **ChatContext** wraps `useChat` from Vercel AI SDK; streams messages from `/api/chat`.
- The two contexts are composed in `src/app/main-content.tsx`, which also manages the resizable panel layout (chat | editor/preview).

### Authentication

JWT tokens (7-day expiry, HTTP-only cookie). Passwords hashed with bcrypt. Users can use the app anonymously — authentication only gates project persistence. Server actions in `src/actions/index.ts` handle sign-up and sign-in.

### Database

SQLite via Prisma. Two models: `User` and `Project`. `Project.data` stores the serialized virtual FS as JSON; `Project.messages` stores chat history as JSON.

### Preview isolation

The preview runs inside a sandboxed `<iframe>`. The `PreviewFrame` component builds an ESM import map so `react`, `react-dom`, and other deps resolve correctly inside the iframe without bundling.

### AI provider

`src/lib/provider.ts` exports a `createModel()` function. When `ANTHROPIC_API_KEY` is set it returns a real Claude model (default `claude-haiku-4-5`); otherwise it returns a mock that streams a static placeholder response — useful for local development without an API key.

The system prompt in `src/lib/prompts/` is sent with `cacheControl: { type: "ephemeral" }` to enable Anthropic prompt caching.

### Testing

Vitest with jsdom + React Testing Library. Tests cover chat components, editor/FileTree, and utility functions. Path aliases (`@/`) are configured in `vitest.config.mts` to match `tsconfig.json`.
