# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
# Initial setup (install deps, generate Prisma client, run migrations)
npm run setup

# Development
npm run dev          # Start dev server with turbopack at http://localhost:3000
npm run dev:daemon   # Start in background, logs to logs.txt

# Build & production
npm run build
npm run start

# Linting
npm run lint

# Tests
npm test                              # Run all tests
npx vitest run src/path/to/file.test.tsx  # Run a single test file

# Database
npx prisma migrate dev   # Apply new migrations
npm run db:reset         # Reset DB (destructive)
npx prisma studio        # Open Prisma Studio GUI
```

The `NODE_OPTIONS='--require ./node-compat.cjs'` prefix is baked into all npm scripts — it patches Node.js compatibility for certain packages. Do not remove it.

Set `ANTHROPIC_API_KEY` in `.env` to enable real AI generation. Without it, the app falls back to `MockLanguageModel` (deterministic stub responses).

## Architecture

UIGen is an AI-powered React component generator. Users describe components in a chat interface; Claude generates code via tool calls; a live preview renders the result in real-time.

### Data Flow

1. User sends a message → `/api/chat` (POST, streaming)
2. Server calls Claude via Vercel AI SDK (`streamText`) with two tools: `str_replace_editor` and `file_manager`
3. Claude responds with tool calls that create/modify virtual files
4. Tool results are streamed back to the client
5. `FileSystemContext` applies tool results to the in-memory virtual file system
6. `PreviewFrame` uses Babel Standalone to transpile JSX and renders the component

### Key Abstractions

**Virtual File System** (`src/lib/file-system.ts`)
In-memory, serializable file system. No disk writes. File contents are plain strings. Serialized as JSON and passed between client/server in project save operations.

**AI Tools** (`src/lib/tools/`)
- `str_replace_editor` — create files, replace text, insert lines
- `file_manager` — rename and delete files

**Language Model Provider** (`src/lib/provider.ts`)
Returns an Anthropic model or `MockLanguageModel`. The mock implements `LanguageModelV1` and generates deterministic component code via tool calls.

**Contexts** (`src/lib/contexts/`)
- `FileSystemContext` — owns the virtual FS state; exposes file CRUD; re-serializes on every change
- `ChatContext` — owns chat messages, project ID, and triggers save to DB

### App Router Layout

```
/                    → Home page; redirects to last project or renders MainContent
/[projectId]         → Project page; loads project from DB and hydrates contexts
/api/chat            → Streaming POST endpoint; only requires auth for project saves
```

`MainContent` (`src/app/main-content.tsx`) renders two resizable panels (react-resizable-panels):
- Left: `ChatInterface`
- Right: `PreviewFrame` (default) or Code view (`FileTree` + `CodeEditor`)

### Authentication

JWT-based sessions stored in httpOnly cookies (no NextAuth). Core logic in `src/lib/auth.ts` (jose for JWT, bcrypt for passwords). Server actions in `src/actions/index.ts`. Middleware at `src/middleware.ts` protects the `/api/chat` route only for project save operations — anonymous users can still generate components.

### Database

Prisma + SQLite. Schema has two models: `User` and `Project`. `Project.messages` and `Project.data` are JSON columns storing serialized chat history and virtual file system state respectively.
