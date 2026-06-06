# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

UIGen is an AI-powered React component generator with live preview. Users describe components in a chat interface, Claude generates React code via structured tool calls, and a sandboxed iframe renders the result in real-time. Built with Next.js 15 (App Router), React 19, TypeScript, Tailwind CSS v4, and the Vercel AI SDK with Anthropic's Claude.

## Commands

```bash
npm run dev          # Start dev server (Turbopack)
npm run build        # Production build
npm run lint         # ESLint
npm run test         # Run all tests (Vitest)
npx vitest run src/lib/__tests__/file-system.test.ts  # Run a single test file
npm run setup        # Install deps + Prisma generate + migrate
npm run db:reset     # Reset database (destructive)
```

## Architecture

### Core Flow

1. User sends a chat message → POST `/api/chat` with messages + virtual file system state
2. The API route streams Claude's response using `streamText` (Vercel AI SDK)
3. Claude modifies files through two tools: `str_replace_editor` (create/edit/view files) and `file_manager` (rename/delete)
4. `FileSystemContext` intercepts tool calls client-side and updates the in-memory `VirtualFileSystem`
5. `PreviewFrame` detects changes, transforms JSX via `@babel/standalone`, builds an import map (esm.sh CDN), and renders in a sandboxed iframe
6. If authenticated, project state (messages JSON + file system JSON) is saved to SQLite via Prisma

### Key Modules

- **`src/app/api/chat/route.ts`** — Chat API endpoint. Configures AI tools, system prompt (with ephemeral caching), max tokens (10k), max steps (40). Saves project on completion.
- **`src/lib/provider.ts`** — Returns `anthropic("claude-haiku-4-5")` if `ANTHROPIC_API_KEY` is set, otherwise a `MockLanguageModel` that streams canned component responses.
- **`src/lib/file-system.ts`** — `VirtualFileSystem` class. In-memory file tree with serialize/deserialize. No disk I/O.
- **`src/lib/transform/jsx-transformer.ts`** — Babel JSX transformation, import map generation, and preview HTML creation for the iframe.
- **`src/lib/prompts/generation.tsx`** — System prompt instructing Claude to generate React + Tailwind components with `/App.jsx` as the root entry point.
- **`src/lib/tools/str-replace.ts`** and **`file-manager.ts`** — Zod-validated AI tool definitions for file operations.
- **`src/lib/contexts/chat-context.tsx`** — Wraps Vercel AI SDK's `useChat`, handles tool call routing to FileSystemContext.
- **`src/lib/contexts/file-system-context.tsx`** — React context managing virtual FS state, selected file, and AI tool execution.

### Auth & Data

- JWT sessions via `jose` with HTTP-only cookies (`src/lib/auth.ts`)
- Prisma + SQLite (`prisma/schema.prisma`). Prisma client generated to `src/generated/prisma/`.
- `Project.messages` and `Project.data` are serialized JSON strings (chat history and file system state)
- Anonymous users get sessionStorage-based persistence (`src/lib/anon-work-tracker.ts`), lost on browser close
- Server actions in `src/actions/` handle auth (signUp/signIn/signOut) and project CRUD

### UI Layout

Three resizable panels (`react-resizable-panels`): chat (left 35%), and code editor + preview (right 65%) with tab switching. Monaco Editor for code editing, markdown rendering in chat.

## Environment Variables

- `ANTHROPIC_API_KEY` — Required for real AI. Without it, the mock provider is used.
- `JWT_SECRET` — Optional in dev (falls back to `"development-secret-key"`)

## Testing

- Vitest with jsdom environment, React Testing Library
- Tests colocated in `__tests__/` directories next to source
- Mocking with `vi.mock()`
- Path alias `@/*` → `./src/*` available in tests via `vite-tsconfig-paths`

## Conventions

- Next.js App Router with server/client component split
- Tailwind CSS v4 (PostCSS plugin: `@tailwindcss/postcss`)
- UI primitives built on Radix UI (`src/components/ui/`)
- `cn()` utility from `src/lib/utils.ts` for merging Tailwind classes (clsx + tailwind-merge)
- AI-generated components always use `/App.jsx` as root, `@/` import alias for local imports, Tailwind for styling
