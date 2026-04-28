# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Is

UIGen is an AI-powered React component generator with live preview. Users describe components in natural language; Claude generates them in real-time via a streaming API. All file operations happen in an in-memory virtual file system — nothing is written to disk during generation.

## Commands

```bash
npm run setup          # First-time: install deps + prisma generate + migrate DB
npm run dev            # Dev server with Turbopack
npm run build          # Production build
npm run lint           # ESLint
npm run test           # Vitest (all tests)
npm run db:reset       # Force-reset Prisma migrations
```

All commands require `NODE_OPTIONS='--require ./node-compat.cjs'` (already embedded in npm scripts).

Environment: create `.env` with `ANTHROPIC_API_KEY=...`. Without it, the app falls back to a `MockLanguageModel` that generates static components.

Run a single test file: `npx vitest src/lib/__tests__/file-system.test.ts`

## Architecture

### Core Abstraction: Virtual File System

`src/lib/file-system.ts` — an in-memory Map of `FileNode` objects. All AI tool calls (create/edit/rename/delete) operate on this. It serializes to/from JSON for API round-trips and DB persistence.

### Data Flow: Chat → AI → Files → Preview

1. **User sends message** → `ChatProvider` (`src/lib/contexts/chat-context.tsx`) calls `/api/chat` via `@ai-sdk/react` `useChat`
2. **API route** (`src/app/api/chat/route.ts`) deserializes the VirtualFileSystem, builds the system prompt with cache control, then calls `streamText()` (Vercel AI SDK) with two tools:
   - `str_replace_editor` — view/create/str_replace/insert on virtual files
   - `file_manager` — rename/delete virtual files
3. **Client-side tool handling** — `onToolCall` in ChatProvider delegates to `useFileSystem().handleToolCall()`, which mutates VirtualFileSystem state and fires `refreshTrigger`
4. **On finish** — API route saves messages + serialized file state as JSON to SQLite via Prisma
5. **Preview** — `PreviewFrame` (`src/components/preview/PreviewFrame.tsx`) Babel-transforms each JSX/TSX file, creates an import map (local files → blob URLs, third-party → esm.sh CDN), and sets `iframe.srcdoc`

### Language Model Provider

`src/lib/provider.ts` exports `getLanguageModel()`. If `ANTHROPIC_API_KEY` is set, returns `anthropic("claude-haiku-4-5")`; otherwise returns `MockLanguageModel`. Both implement `LanguageModelV1`.

### Auth

- JWT (HS256) stored in HTTPOnly cookie, 7-day expiry — `src/lib/auth.ts`
- Server actions in `src/actions/index.ts`: signUp/signIn hash passwords with bcrypt, call `createSession()`
- Middleware (`src/middleware.ts`) protects `/api/projects` and `/api/filesystem` routes
- `JWT_SECRET` defaults to `"development-secret-key"` — override in production

### Database

Prisma + SQLite (`prisma/dev.db`). Schema is the source of truth — see `prisma/schema.prisma` for all models and field definitions.

### System Prompt Constraints

`src/lib/prompts/generation.tsx` tells Claude:
- Every project must have `/App.jsx` as the root entry point
- Use Tailwind CSS for styling
- Use `@/` alias for non-library imports within the VFS
- No HTML files — React only

## Code Style

**Comments**: Use sparingly — only for complex or non-obvious logic. Do not comment self-explanatory code.

## Key Patterns

**Path alias**: `@/*` maps to `src/*` (tsconfig + vitest).

**Serialization**: `VirtualFileSystem.serialize()` → JSON string → stored in `Project.data`. Deserialized via `VirtualFileSystem.deserializeFromNodes()` on each API call.

**Refresh cycle**: File mutations set `refreshTrigger` in `FileSystemContext` → `FileTree`, `CodeEditor`, and `PreviewFrame` all re-render.
