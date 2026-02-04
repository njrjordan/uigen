# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

UIGen is an AI-powered React component generator with live preview. It uses Claude (Anthropic) to generate React components through a chat interface and displays them in real-time using a virtual file system. Components are generated in-memory without writing to disk.

## Commands

### Setup
```bash
npm run setup
```
Installs dependencies, generates Prisma client, and runs database migrations. Run this first when setting up the project.

### Development
```bash
npm run dev
```
Starts Next.js development server with Turbopack on http://localhost:3000. Uses Node compatibility layer via node-compat.cjs.

### Testing
```bash
npm test
```
Runs Vitest tests in watch mode with jsdom environment.

### Database
```bash
npm run db:reset
```
Resets the SQLite database and runs migrations. Use with caution as it deletes all data.

### Build & Deploy
```bash
npm run build   # Production build
npm start       # Start production server
npm run lint    # Run ESLint
```

## Architecture

### Virtual File System (VFS)

The core of UIGen is its in-memory virtual file system (`src/lib/file-system.ts`). This VFS:
- Stores all generated files in a tree structure using `Map<string, FileNode>`
- Never writes files to disk - everything is ephemeral
- Provides file operations: create, read, update, delete, rename
- Serializes to/from JSON for persistence in the database
- Implements text editor commands (view, str_replace, insert) used by the AI agent

The VFS is reconstructed on every chat API request from serialized data stored in the database.

### AI Component Generation Flow

1. **Chat API Route** (`src/app/api/chat/route.ts`):
   - Receives messages and serialized VFS state from the client
   - Reconstructs VFS from serialized data
   - Uses Vercel AI SDK with Anthropic's Claude (claude-haiku-4-5)
   - Provides two tools to the AI agent:
     - `str_replace_editor`: Create/view/edit files using text operations
     - `file_manager`: Rename/delete files and directories
   - AI uses these tools to generate component files
   - On completion, saves updated VFS state and messages to database (for authenticated users)

2. **Mock Provider Fallback** (`src/lib/provider.ts`):
   - When `ANTHROPIC_API_KEY` is not set, uses `MockLanguageModel`
   - Returns static counter/form/card components based on keywords
   - Simulates multi-step tool usage for consistent behavior
   - Limited to 4 steps vs 40 for real provider to prevent repetition

3. **JSX Transformation** (`src/lib/transform/jsx-transformer.ts`):
   - Transforms TypeScript/JSX to browser-compatible ES modules using Babel
   - Creates import maps to resolve module paths
   - Generates blob URLs for transformed modules
   - Handles CSS imports separately, collecting them into inline styles
   - Supports `@/` path alias (maps to root `/`)
   - Provides placeholder modules for missing imports
   - Returns syntax errors for display in preview

4. **Live Preview** (`src/components/preview/`):
   - Renders transformed components in an iframe
   - Uses ES module import maps for module resolution
   - Includes Tailwind CSS via CDN
   - Shows error boundaries for runtime errors
   - Displays syntax errors with formatted error messages
   - Entry point is always `/App.jsx` or `/App.tsx`

### Authentication & Sessions

- JWT-based authentication (`src/lib/auth.ts`) using jose library
- Session stored in HTTP-only cookies with 7-day expiration
- Anonymous users can use the app but projects are not persisted
- Authenticated users have projects saved to SQLite database
- Middleware (`src/middleware.ts`) handles auth state

### Database Schema

Prisma with SQLite (`prisma/schema.prisma`):
- **User**: id, email, hashed password (bcrypt)
- **Project**: id, name, userId (nullable), messages (JSON), data (JSON - serialized VFS)
- Projects can be anonymous (userId=null) or owned by authenticated users
- Prisma client generated to `src/generated/prisma/`

### Project State Management

Projects contain two key pieces of state:
1. **messages**: Full chat conversation history (JSON array)
2. **data**: Serialized VFS state (JSON object with file paths as keys)

When loading a project:
- Deserialize VFS from `data` field
- Load messages into chat interface
- Preview automatically renders `/App.jsx` from VFS

### Component Structure

- **Chat Components** (`src/components/chat/`): Message list, input, markdown rendering
- **Preview Components** (`src/components/preview/`): Iframe-based component preview
- **Editor Components** (`src/components/editor/`): Monaco-based code editor for viewing/editing generated files
- **UI Components** (`src/components/ui/`): Radix UI primitives (button, dialog, tabs, etc.)

### File Path Conventions

- All paths in VFS are absolute (start with `/`)
- Path alias `@/` maps to root `/` directory
- Component files typically in `/components/` directory
- Entry point is always `/App.jsx` or `/App.tsx`
- CSS files supported but rendered inline via style tag

## Key Implementation Details

- **AI Tools**: The AI agent uses `str_replace_editor` for most file operations. It can view files with line numbers, create files with automatic parent directory creation, and perform string replacements.

- **Import Resolution**: The JSX transformer creates comprehensive import maps supporting:
  - Absolute paths (`/components/Foo`)
  - Path aliases (`@/components/Foo`)
  - Extensions optional (`.jsx`, `.tsx` automatically resolved)
  - Third-party packages via esm.sh CDN

- **No Build Step for Preview**: Preview uses native ES modules in the browser with import maps. No bundler required for runtime.

- **Anonymous Work Tracking** (`src/lib/anon-work-tracker.ts`): Tracks work duration for anonymous users to potentially prompt sign-up.

## Testing

Tests use Vitest with React Testing Library and jsdom:
- Component tests in `__tests__` directories adjacent to components
- File system tests in `src/lib/__tests__/`
- Transformer tests in `src/lib/transform/__tests__/`

## Environment Variables

- `ANTHROPIC_API_KEY`: Required for AI generation (optional - uses mock provider if not set)
- `JWT_SECRET`: Used for session signing (defaults to "development-secret-key")
- `NODE_ENV`: Set to "production" for secure cookies

## Tech Stack

- **Framework**: Next.js 15 (App Router)
- **React**: v19 with automatic JSX runtime
- **TypeScript**: Strict mode
- **Styling**: Tailwind CSS v4 (via PostCSS)
- **Database**: Prisma with SQLite
- **AI**: Anthropic Claude via Vercel AI SDK
- **Code Editor**: Monaco Editor (VS Code editor in browser)
- **JSX Transform**: Babel Standalone
- **UI Components**: Radix UI primitives
