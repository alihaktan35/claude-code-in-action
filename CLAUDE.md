# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

UIGen is an AI-powered React component generator with live preview. Users describe components in a chat interface; Claude generates the code using a virtual file system (no disk writes). Components can be previewed live and exported.

## Setup & Commands

```bash
npm run setup        # Install deps, generate Prisma client, run migrations (first-time setup)
npm run dev          # Start dev server with Turbopack
npm run build        # Production build
npm run lint         # ESLint
npm run test         # Run all tests with vitest
npm run db:reset     # Reset SQLite database
```

Run a single test file:
```bash
npx vitest run src/lib/__tests__/file-system.test.ts
```

**Environment:** Copy `.env.example` to `.env` and set `ANTHROPIC_API_KEY`. If absent, the app uses a mock provider that returns pre-built component examples.

## Architecture

### Request Flow
1. User sends a chat message → `POST /api/chat`
2. `route.ts` calls `streamText` (Vercel AI SDK) with the system prompt and two tools
3. Claude invokes `str_replace_editor` (view/create/edit files) and `file_manager` (rename/delete)
4. Tool calls mutate a `VirtualFileSystem` instance passed in the request body
5. On stream completion, chat history + file state are persisted to SQLite via Prisma
6. The frontend re-renders the preview by transpiling files with Babel standalone

### Virtual File System (`src/lib/file-system.ts`)
In-memory FS; serialized as JSON and sent with every chat request so the AI always has the current state. `/App.jsx` is the required root entry point for the preview renderer.

### AI Provider (`src/lib/provider.ts`)
Wraps `claude-haiku-4-5` via `@ai-sdk/anthropic`. Falls back to `MockLanguageModel` (implements `LanguageModelV1`) when no API key is set — useful for local dev without billing.

### Authentication (`src/lib/auth.ts`, `src/middleware.ts`)
JWT via `jose`, stored in `auth-token` cookie (7-day sessions). Anonymous users can use the app; authenticated users get project persistence. Middleware protects `/api/projects` and `/api/filesystem`.

### Data Model
- **User** → has many **Project**
- **Project** stores `messages` (JSON string) and `data` (JSON string of the virtual FS)
- SQLite via Prisma; schema at `prisma/schema.prisma`

### Frontend Layout (`src/app/main-content.tsx`)
Three-panel resizable layout: Chat (left) | Editor/Preview (right). The right panel switches between Preview (live Babel transpilation) and Code (Monaco editor + file tree) via tabs.

### Key Paths
- `src/app/api/chat/route.ts` — core AI streaming endpoint
- `src/lib/prompts/generation.tsx` — system prompt sent to Claude
- `src/lib/tools/` — Zod-validated tool definitions for the AI
- `src/lib/transform/` — JSX/Babel transformation for live preview
- `src/components/preview/` — iframe-based preview renderer
- `src/lib/contexts/` — `FileSystemContext` and `ChatContext` shared state

## Tech Stack Notes
- **Next.js 15** App Router with React 19; use server components and server actions where appropriate
- **Tailwind CSS v4** (not v3) — config is in `postcss.config.mjs`, not `tailwind.config.js`
- **Shadcn UI** (new-york style, neutral base) — add components via `npx shadcn@latest add <component>`
- Path alias `@/` maps to `src/`
- Tests use **vitest** + **jsdom** + **@testing-library/react**
