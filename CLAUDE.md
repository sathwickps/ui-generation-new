    # CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
npm run setup       # Install deps, generate Prisma client, run migrations
npm run dev         # Start dev server (Turbopack)
npm run build       # Production build
npm run lint        # ESLint
npm run test        # Run Vitest test suite
npm run db:reset    # Force reset SQLite database (destructive)
```

Single test file: `npx vitest run src/lib/__tests__/file-system.test.ts`

**Required env:** Copy `.env.example` to `.env`. `ANTHROPIC_API_KEY` is optional — without it the app uses `MockLanguageModel` (deterministic templates, no API cost). `JWT_SECRET` defaults to `"development-secret-key"` if unset.

## Architecture

UIGen is a Next.js 15 full-stack app that generates React components via Claude AI with a live preview.

### Core Data Flow

1. **User sends a message** → `ChatInterface` → POST `/api/chat`
2. **Claude (or mock) responds** with tool calls (`str_replace_editor`, `file_manager`)
3. **Tool handlers** in `FileSystemContext` update the in-memory `VirtualFileSystem`
4. **`PreviewFrame`** picks up changes, transforms JSX→JS via Babel standalone, and renders in a sandboxed iframe using an import map (local files → blob URLs, external packages → esm.sh CDN)
5. **On completion**, if authenticated, messages + serialized VirtualFileSystem are saved to SQLite via Prisma

### Key Modules

| Path | Role |
|---|---|
| `src/app/api/chat/route.ts` | Claude API endpoint — streams via Vercel AI SDK, wires tools, persists to DB |
| `src/lib/file-system.ts` | `VirtualFileSystem` — in-memory Map-based file tree, serializable to JSON |
| `src/lib/provider.ts` | Returns Anthropic or `MockLanguageModel`; mock is used when no API key |
| `src/lib/transform/jsx-transformer.ts` | Babel JSX→JS, `createImportMap`, `createPreviewHTML` |
| `src/lib/contexts/file-system-context.tsx` | React context wrapping VirtualFileSystem; exposes tool handlers to chat |
| `src/lib/contexts/chat-context.tsx` | Vercel AI SDK `useChat` wiring; passes serialized FS state to API |
| `src/components/preview/PreviewFrame.tsx` | Live iframe preview; rebuilds `srcdoc` on every FS change |
| `src/actions/index.ts` | Server actions: `signUp`, `signIn`, `signOut`, `getUser` |
| `src/lib/auth.ts` | JWT sessions via `jose`, stored in HTTP-only `auth-token` cookie |
| `prisma/schema.prisma` | SQLite schema: `User` and `Project` (messages + FS state stored as JSON strings) |

### State Management

- **`FileSystemContext`** owns the virtual file system and the two Claude tool handlers. Components call `useFileSystem()`.
- **`ChatContext`** owns message history (Vercel AI SDK). Components call `useChat()`.
- Contexts are composed in `FileSystemProvider` → `ChatProvider` in the root layout.

### Preview Sandboxing

The iframe uses `sandbox="allow-scripts allow-same-origin allow-forms"` with `srcdoc` (not `src`) to avoid CORS. All local VFS files are turned into blob URLs at render time; npm imports resolve against `esm.sh`.

### Authentication

JWT with 7-day expiry. `middleware.ts` guards `/api/projects` and `/api/filesystem`. Anonymous projects (`userId: null`) are allowed — a project becomes owned when a user signs in.

### Mock Provider

`MockLanguageModel` in `src/lib/provider.ts` generates deterministic components (Counter, ContactForm, Card) based on keyword matching. It simulates 4 agentic steps with delays and is used automatically when `ANTHROPIC_API_KEY` is absent.

### Testing

Tests use Vitest with the jsdom environment (`vitest.config.mts`). Coverage spans `VirtualFileSystem`, contexts, JSX transformer, and chat/editor components.
