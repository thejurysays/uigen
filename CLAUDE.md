# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
npm run setup       # First-time setup: install deps, generate Prisma client, run migrations
npm run dev         # Start dev server with Turbopack
npm run build       # Production build
npm run lint        # ESLint via Next.js
npm run test        # Run Vitest tests
npm run db:reset    # Reset SQLite database
```

Run a single test file: `npx vitest run src/path/to/test.ts`

**Environment:** Copy `.env.example` to `.env` and set `ANTHROPIC_API_KEY`. The app falls back to `MockLanguageModel` if the key is absent.

## Architecture

UIGen is a Next.js 15 App Router app that streams Claude-generated React components into a live in-browser preview. There is no actual disk-based component file system — all generated files live in a `VirtualFileSystem` instance in memory.

### Request flow

1. User submits a prompt in the chat UI
2. `POST /api/chat` (`src/app/api/chat/route.ts`) streams a response using Vercel AI SDK's `streamText`
3. Claude receives the system prompt (`src/lib/prompts/generation.tsx`) and two tools:
   - `str_replace_editor` — create/view/edit virtual files
   - `file_manager` — rename/delete virtual files
4. Tool call results update the `VirtualFileSystem` (`src/lib/file-system.ts`) held in React context (`src/lib/contexts/FileSystemContext`)
5. The JSX transformer (`src/lib/transform/jsx-transformer.ts`) converts virtual file contents to executable code
6. `PreviewFrame` (`src/components/preview/`) renders an `<iframe>` that transpiles and executes the code via Babel standalone

### Key modules

| Path | Role |
|---|---|
| `src/lib/file-system.ts` | `VirtualFileSystem` class — in-memory file store |
| `src/lib/provider.ts` | Returns Anthropic model or `MockLanguageModel` based on env |
| `src/lib/prompts/generation.tsx` | System prompt sent to Claude; controls generation behavior |
| `src/lib/tools/` | `str_replace_editor` and `file_manager` tool definitions |
| `src/lib/transform/jsx-transformer.ts` | Transforms JSX to preview-ready HTML |
| `src/lib/contexts/` | `ChatContext` and `FileSystemContext` React contexts |
| `src/actions/` | Next.js Server Actions for project CRUD |
| `src/lib/auth.ts` | JWT session auth (bcrypt passwords); anonymous mode supported |
| `prisma/schema.prisma` | SQLite schema: `User` + `Project` (stores serialized messages/files) |

### Path alias

`@/*` maps to `./src/*` (configured in `tsconfig.json`).

### Prompt caching

The system prompt uses `ephemeral` cache control (Anthropic SDK feature) to reduce latency on repeated requests. Keep the system prompt stable across turns to maximize cache hits.
