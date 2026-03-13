# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What is UIGen

An AI-powered React component generator with live preview. Users describe components in a chat interface, Claude generates React code via tool calls that manipulate a virtual file system, and the result is rendered live in an iframe.

Works without an Anthropic API key (falls back to a mock provider returning static components).

## Commands

```bash
npm run setup          # Install deps, generate Prisma client, run migrations
npm run dev            # Dev server with Turbopack on http://localhost:3000
npm run build          # Production build
npm run lint           # ESLint (extends next)
npm test               # Vitest (all tests)
npx vitest run <path>  # Run a single test file
npm run db:reset       # Reset SQLite database
```

The dev server requires `NODE_OPTIONS='--require ./node-compat.cjs'` (already wired into npm scripts).

## Architecture

### Request Flow

1. User sends a message via `ChatProvider` (wraps `@ai-sdk/react` `useChat`)
2. `POST /api/chat` receives messages + serialized virtual file system
3. The route prepends a system prompt (`src/lib/prompts/generation.tsx`) and calls `streamText` with two tools: `str_replace_editor` and `file_manager`
4. Claude generates code by calling these tools, which mutate a server-side `VirtualFileSystem` instance
5. Tool calls stream back to the client; `FileSystemProvider.handleToolCall` applies the same mutations to the client-side VFS
6. `PreviewFrame` reacts to file system changes, transforms JSX via `@babel/standalone`, creates blob URLs + an import map, and renders everything in a sandboxed iframe

### Key Abstractions

- **VirtualFileSystem** (`src/lib/file-system.ts`) — In-memory file system with tree structure using `Map<string, FileNode>`. Serializable to/from JSON for transport between client and server. No disk I/O.
- **AI Tools** (`src/lib/tools/`) — Two Vercel AI SDK tools the LLM can call:
  - `str_replace_editor`: view, create, str_replace, insert operations on virtual files
  - `file_manager`: rename and delete operations
- **JSX Transformer** (`src/lib/transform/jsx-transformer.ts`) — Client-side Babel transformation that builds import maps, resolves `@/` aliases, handles CSS imports, creates placeholder modules for missing imports, and loads third-party packages from esm.sh
- **Provider** (`src/lib/provider.ts`) — Returns either the real Anthropic model (`claude-haiku-4-5`) or a `MockLanguageModel` when no API key is set
- **Contexts** — `FileSystemProvider` and `ChatProvider` are the two main React contexts, nested in `MainContent`

### Data Model (Prisma + SQLite)

Database schema is defined in `prisma/schema.prisma` — refer to it for the authoritative data structure.

- `User`: email/password auth with bcrypt + JWT sessions (jose)
- `Project`: stores serialized messages and virtual file system data as JSON strings. Optional `userId` (anonymous users can use the app without an account)

### Auth

JWT-based sessions stored in httpOnly cookies. `src/lib/auth.ts` handles session creation/verification. Server actions in `src/actions/index.ts` handle signUp/signIn/signOut. Middleware protects `/api/projects` and `/api/filesystem` routes.

### Pages

- `/` — Anonymous users see `MainContent` directly; authenticated users redirect to their most recent project (or create one)
- `/[projectId]` — Loads a saved project with its messages and file system state

## Code Style

- Use comments sparingly, only for complex code

## Tech Stack

- Next.js 15 (App Router, Turbopack), React 19, TypeScript
- Tailwind CSS v4, shadcn/ui (new-york style, `@/components/ui/`)
- Prisma with SQLite (`prisma/dev.db`), Prisma client output at `src/generated/prisma`
- Vercel AI SDK (`ai` + `@ai-sdk/anthropic`), `@babel/standalone` for client-side JSX transform
- Vitest + jsdom + React Testing Library for tests
- Path alias: `@/*` maps to `./src/*`
