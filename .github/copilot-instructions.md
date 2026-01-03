# Copilot Instructions for Plannotator

## Architecture Overview

Plannotator is a **Claude Code plugin** that intercepts plan approval via hooks, serves a browser-based annotation UI, and returns structured feedback. The system has three distinct architectural layers:

1. **Hook Server** (`apps/hook/server/index.ts`) - Ephemeral Bun server that reads stdin, serves UI, handles decisions
2. **UI Layer** (`packages/ui/`) - Shared React components, utilities, and hooks used by all apps
3. **Apps** (`apps/`) - Three separate Vite builds: hook (single-file HTML), portal (share.plannotator.ai), marketing

### Key Architectural Decisions

**Why cookies over localStorage?** Each hook invocation runs on a random port. localStorage is scoped by `origin:port`, but cookies are scoped by domain only. This allows settings to persist across different hook sessions on `localhost:*`.

**Why single-file HTML?** The hook app uses `vite-plugin-singlefile` to bundle everything (JS, CSS, assets) into one `dist/index.html`. This simplifies deployment - the server just serves one embedded file via `import indexHtml from "../dist/index.html" with { type: "text" }`.

**Why Bun?** Hook server uses Bun for fast startup, built-in `Bun.stdin.text()`, and `Bun.write()` for file operations. The `$` template tag runs shell commands.

**Why web-highlighter?** Standard DOM selections don't work well across block elements. `web-highlighter` stores selections as `startMeta`/`endMeta` with `{parentTagName, parentIndex, textOffset}` to reconstruct highlights after re-render. Code blocks use manual `<mark>` since web-highlighter can't select inside `<pre>`.

## Project Structure

```
plannotator/
├── apps/
│   ├── hook/          # Claude Code plugin (single-file build)
│   │   ├── server/    # Bun server reads stdin, serves UI, writes plan files
│   │   ├── dist/      # Built index.html embedded in server
│   │   └── hooks/     # hooks.json defines ExitPlanMode matcher
│   ├── portal/        # Share UI at share.plannotator.ai
│   ├── marketing/     # Main site at plannotator.ai
│   └── opencode-plugin/ # OpenCode integration (built from hook dist)
└── packages/
    ├── ui/            # Shared components, utils, hooks, types
    └── editor/        # Main App.tsx (used by hook, portal, marketing)
```

## Development Workflow

### Build Commands
```bash
bun install           # Install dependencies (uses workspace: protocol)
bun run build:hook    # Build hook app to single HTML file
bun run build:portal  # Build share.plannotator.ai
bun run build         # Build hook + opencode-plugin
bun run dev:hook      # Dev server on :3000
```

### Testing the Hook Flow
```bash
# Method 1: Test server directly with mock stdin
echo '{"tool_input":{"plan":"# Test Plan\nContent here"}}' | bun run --cwd apps/hook serve

# Method 2: Install plugin locally
claude --plugin-dir ./apps/hook
```

### Important: vite-plugin-singlefile
After building, verify `apps/hook/dist/index.html` exists. The server imports this at compile time, not runtime. If you change UI code, you must rebuild the hook app.

## Code Conventions

### Path Handling (Security Critical)
Always use Node.js `path` module for cross-platform compatibility. Validate user-provided paths to prevent directory traversal:
```typescript
const repoRoot = path.resolve(process.cwd());
const absolutePath = path.resolve(repoRoot, userPath);
const relativePath = path.relative(repoRoot, absolutePath);
const isInRepo = relativePath && !relativePath.startsWith('..') && !path.isAbsolute(relativePath);
```

### State Management Pattern
The app uses React hooks with URL-based sharing. Key pattern:
1. `useSharing` hook manages markdown, annotations, savePath, systemPrompt
2. On mount, check for URL hash → decompress → restore state
3. On state change, compress state → update share URL
4. Use `storage.ts` (cookies) for preferences that persist across sessions

### API Mode vs Standalone
The app runs in two contexts:
- **API mode**: Connected to hook server (`isApiMode = true`). Shows save path/system prompt inputs.
- **Standalone**: Portal/marketing apps. No server connection.

Check via `fetch('/api/plan')` on mount. If successful, in API mode.

### Compression Pipeline (URL Sharing)
```typescript
// Encoding: state → JSON → deflate-raw → base64 → URL-safe base64
const json = JSON.stringify(payload);
const stream = new CompressionStream('deflate-raw');
const base64 = btoa(String.fromCharCode(...compressed));
return base64.replace(/\+/g, '-').replace(/\//g, '_').replace(/=/g, '');

// Decoding: reverse the process
```

## Hook Communication Protocol

The server reads a JSON event from stdin, extracts `tool_input.plan`, serves UI, waits for decision, then outputs JSON to stdout:

**Approve:**
```json
{"hookSpecificOutput":{"decision":{"behavior":"allow"}}}
```

**Deny (Request Changes):**
```json
{"hookSpecificOutput":{"decision":{"behavior":"deny","message":"feedback here"}}}
```

If `savePath` is provided in deny, the server writes the plan to that file before returning.

## Critical Integration Points

### Hook Registration
`apps/hook/hooks/hooks.json` defines the hook:
```json
{
  "hooks": {
    "PermissionRequest": [{
      "matcher": "ExitPlanMode",
      "hooks": [{"type": "command", "command": "plannotator", "timeout": 1800}]
    }]
  }
}
```

### Markdown Parser
`packages/ui/utils/parser.ts` - **parseMarkdownToBlocks()** uses naive line-by-line parsing, not an AST. This ensures predictable text positions for web-highlighter. Don't replace with remark/unified without rethinking the annotation system.

### Server Port Strategy
- **Local**: Random port (`port: 0`) to avoid conflicts
- **SSH**: Fixed port (19432) for port forwarding setup
- Detection: `process.env.SSH_TTY || process.env.SSH_CONNECTION`

## Data Flow

1. Claude calls `ExitPlanMode` → hook fires
2. Server receives JSON via stdin with `tool_input.plan`
3. Server starts on random/fixed port, opens browser
4. UI fetches plan via `/api/plan`
5. User annotates, optionally sets save path + system prompt
6. User clicks "Approve" or "Request Changes"
7. If deny with savePath: server writes file, returns deny decision
8. Server outputs JSON to stdout, exits

## Common Pitfalls

1. **Don't use localStorage** - Use `packages/ui/utils/storage.ts` (cookies)
2. **Empty string validation** - Always trim before checking `if (savePath)` to avoid empty strings in payloads
3. **Cross-platform paths** - Use `path.sep`, `path.resolve()`, not string manipulation
4. **Code block highlighting** - Can't use web-highlighter; manually wrap in `<mark>`
5. **Build before testing** - UI changes require `bun run build:hook` to regenerate embedded HTML

## External Dependencies

- **Bun runtime** - Server execution, file ops, shell commands
- **web-highlighter** - DOM selection persistence (see packages/ui/components/Viewer.tsx)
- **highlight.js** - Syntax highlighting for code blocks
- **vite-plugin-singlefile** - Single HTML bundle for hook app
- **Tailwind CSS v4** - Uses `@tailwindcss/vite` plugin

## Testing Strategy

No automated tests currently. Manual testing via:
1. Mock stdin: `echo '{"tool_input":{"plan":"..."}}' | bun run serve`
2. Local plugin: `claude --plugin-dir ./apps/hook`
3. SSH testing: See `tests/manual/ssh/` for Docker-based SSH scenarios

## License & Contributions

BSL-1.1 licensed. PRs require CLA acceptance (see CONTRIBUTING.md). Do not copy GPL code.
