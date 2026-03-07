# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
# First-time setup
npm run setup          # installs deps, generates Prisma client, runs migrations

# Development
npm run dev            # starts dev server at http://localhost:3000 (turbopack)
npm run dev:daemon     # same but in background, logs to logs.txt

# Build & production
npm run build
npm run start

# Linting
npm run lint

# Tests
npm test               # run all tests (vitest)
npm test -- src/lib/__tests__/file-system.test.ts  # run a single test file

# Database
npm run db:reset       # drop and re-run all migrations (destructive)
npx prisma migrate dev # apply new migrations
npx prisma generate    # regenerate client after schema changes
```

The Prisma client is generated to `src/generated/prisma` (non-standard location — set in `prisma/schema.prisma`).

## Environment

Copy `.env.example` or create `.env`:
- `ANTHROPIC_API_KEY` — optional. Without it, a `MockLanguageModel` in `src/lib/provider.ts` returns static responses instead of calling Claude.
- `JWT_SECRET` — optional. Defaults to `"development-secret-key"` in dev.

## Architecture

### High-level flow

1. User types in chat → `POST /api/chat` receives messages + serialized virtual file system
2. The API route streams AI responses via Vercel AI SDK (`streamText`), giving Claude two tools: `str_replace_editor` and `file_manager`
3. Tool calls stream back to the client; `FileSystemContext` (`handleToolCall`) intercepts them and applies mutations to the in-memory `VirtualFileSystem`
4. `PreviewFrame` watches the file system and re-renders by transforming JSX with Babel standalone in the browser, building an ES module import map with blob URLs, and loading it in an `<iframe>`
5. On finish, if the user is authenticated and a `projectId` exists, the API route saves messages + serialized file system to the database

### Key modules

| Path | Purpose |
|---|---|
| `src/lib/file-system.ts` | `VirtualFileSystem` — in-memory tree, no disk I/O. Serializes to/from plain JSON for persistence and API transport. |
| `src/lib/contexts/file-system-context.tsx` | React context wrapping `VirtualFileSystem`. Exposes `handleToolCall` which routes `str_replace_editor` / `file_manager` tool calls to VFS mutations. |
| `src/lib/contexts/chat-context.tsx` | Manages chat messages and calls `/api/chat` via Vercel AI SDK `useChat`. |
| `src/lib/transform/jsx-transformer.ts` | Client-side JSX pipeline: Babel transform → blob URLs → ES import map. Resolves `@/` path aliases, strips CSS imports, creates placeholder modules for missing local imports, fetches third-party packages from `esm.sh`. |
| `src/components/preview/PreviewFrame.tsx` | Renders an `<iframe>` using the HTML produced by `createPreviewHTML`. Displays compile errors inline. |
| `src/app/api/chat/route.ts` | The only API route that uses Claude. Reconstructs `VirtualFileSystem` from request, streams AI response with tools, persists on finish. |
| `src/lib/provider.ts` | Returns `anthropic("claude-haiku-4-5")` when API key is present, otherwise `MockLanguageModel`. |
| `src/lib/tools/str-replace.ts` | Builds the `str_replace_editor` tool (create / str_replace / insert / view commands). |
| `src/lib/tools/file-manager.ts` | Builds the `file_manager` tool (rename / delete commands). |
| `src/lib/prompts/generation.tsx` | System prompt for the component generation AI. |
| `src/lib/auth.ts` | JWT sessions via `jose`, stored in an httpOnly cookie (`auth-token`). Server-only. |
| `src/lib/prisma.ts` | Prisma client singleton. |
| `src/actions/` | Server actions for project CRUD (`create-project`, `get-project`, `get-projects`). |
| `src/middleware.ts` | Protects `/api/projects` and `/api/filesystem` routes; `/api/chat` is intentionally unprotected. |
| `src/app/main-content.tsx` | Root client component. Lays out the resizable chat + preview/code panels and wraps them in `FileSystemProvider` and `ChatProvider`. |

### Data persistence

- Anonymous users: file system state lives only in React state (lost on refresh).
- Authenticated users: `Project` rows store `messages` (JSON array) and `data` (serialized `VirtualFileSystem` nodes) as text columns in SQLite.

### Preview rendering

The preview uses browser-native ES modules and import maps:
- `App.jsx` (or the first root-level file) is the entry point
- All project files are Babel-transformed and turned into `blob:` URLs at render time
- Third-party imports (non-relative, non-`@/`) are auto-mapped to `https://esm.sh/<package>`
- Tailwind CSS is loaded via CDN script tag in the preview `<iframe>`

### Testing

Tests use Vitest + jsdom + `@testing-library/react`. Test files live alongside source in `__tests__/` subdirectories. The vitest config (`vitest.config.mts`) enables tsconfig path aliases via `vite-tsconfig-paths`.

### `node-compat.cjs`

### use comments sparingly

All `next` commands are run with `NODE_OPTIONS='--require ./node-compat.cjs'`. This shim patches Node.js built-ins for compatibility with Prisma/bcrypt in the Next.js edge/turbopack environment.
