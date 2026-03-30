# LocalPOV

[![npm version](https://img.shields.io/npm/v/localpov)](https://www.npmjs.com/package/localpov)
[![npm downloads](https://img.shields.io/npm/dw/localpov)](https://www.npmjs.com/package/localpov)
[![license](https://img.shields.io/npm/l/localpov)](https://github.com/manish-bhanushali-404/localpov/blob/main/LICENSE)
[![Node.js 18+](https://img.shields.io/badge/Node.js-18%2B-brightgreen)](https://nodejs.org)

Development context bridge for AI coding agents. One MCP server, 14 tools.

- **Terminal capture** — every shell session, auto-captured
- **Browser console** — errors, warnings, unhandled exceptions
- **Network failures** — 4xx/5xx responses, CORS errors, slow requests
- **Build errors** — structured `{file, line, col, message}` from TS, ESLint, Webpack, Vite, Rust, Go
- **Docker logs** — container output without leaving your editor
- **Screenshots** — on-demand browser viewport capture
- **Port scanning** — what's up, what's down
- **System health** — memory, CPU, uptime

Your AI agent sees everything happening in your dev environment — without you copy-pasting anything.

## Quick start

### Claude Code

```bash
claude mcp add localpov -- npx -y localpov --mcp
```

That's it. One command. Restart Claude Code and all 14 tools are available.

Shell integration is installed automatically on first run — restart your terminal to start capturing.

### Claude Desktop

Add to `claude_desktop_config.json`:

- **macOS**: `~/Library/Application Support/Claude/claude_desktop_config.json`
- **Windows**: `%APPDATA%\Claude\claude_desktop_config.json`

```json
{
  "mcpServers": {
    "localpov": {
      "command": "npx",
      "args": ["-y", "localpov", "--mcp"]
    }
  }
}
```

### Cursor

Add to `.cursor/mcp.json` in your project root (or `~/.cursor/mcp.json` for global):

```json
{
  "mcpServers": {
    "localpov": {
      "command": "npx",
      "args": ["-y", "localpov", "--mcp"]
    }
  }
}
```

### VS Code / Copilot

Add to `.vscode/mcp.json` in your project root:

```json
{
  "servers": {
    "localpov": {
      "command": "npx",
      "args": ["-y", "localpov", "--mcp"]
    }
  }
}
```

### Windsurf

Add to your MCP config:

- **macOS/Linux**: `~/.codeium/windsurf/mcp_config.json`
- **Windows**: `%USERPROFILE%\.codeium\windsurf\mcp_config.json`

```json
{
  "mcpServers": {
    "localpov": {
      "command": "npx",
      "args": ["-y", "localpov", "--mcp"]
    }
  }
}
```

Or configure via Windsurf UI: Cascade panel → MCPs icon → Add server.

### Any other MCP client

LocalPOV uses **stdio transport**. Any MCP client that supports stdio works with:

```json
{
  "command": "npx",
  "args": ["-y", "localpov", "--mcp"]
}
```

## Why

AI coding agents are blind. They can read and write files, but they can't see:

- What your terminal just printed
- That the browser console is full of CORS errors
- That `fetch('/api/users')` is returning 500
- That the Docker container crashed 30 seconds ago
- That TypeScript found 12 errors on build

You end up copy-pasting error messages into chat. LocalPOV fixes this — the agent reads your dev environment directly.

## What the AI agent sees

14 tools available via MCP:

| Tool | What it does |
|------|-------------|
| `get_diagnostics` | One-call health check: terminal errors, browser console, network failures, port status, memory |
| `list_sessions` | All captured terminal sessions (PID, shell, alive/dead) |
| `read_terminal` | Last N lines from any terminal session, with pagination |
| `read_command` | Output of a specific command by index (supports negative indexing) |
| `get_errors` | Errors across all sessions (JS, TS, Python, Rust, Go patterns) |
| `search_all` | Regex search across all terminal output |
| `read_browser` | Browser console errors + failed/slow network requests |
| `take_screenshot` | Capture browser viewport |
| `get_build_errors` | Structured build errors: `{file, line, col, message}` |
| `docker` | List containers or read container logs |
| `tail_log` | Read last N lines of any log file |
| `check_ports` | What's listening, what's down |
| `check_env` | Which env vars exist (never exposes values) |
| `process_health` | System memory, CPU, uptime |

## How it works

### Terminal capture

Shell integration hooks into your shell startup (`bash`, `zsh`, `fish`, `powershell`). Every terminal session's output is captured to `~/.localpov/sessions/` as plain text logs. The AI agent reads these via `read_terminal` or `get_errors`.

This is installed automatically when you run `npx -y localpov` for the first time.

### Browser capture (optional)

Start the proxy to capture browser console and network activity:

```bash
npx -y localpov --port 3000
```

Then open `http://localhost:4000` instead of `http://localhost:3000` — your app works exactly the same, but the agent can now see console errors, network failures, and take screenshots.

No browser extension needed. Works in any browser.

### Dashboard

Open `http://localhost:4000/__localpov__/` to see a visual debug panel:

- **Apps** — detected dev servers with one-click preview
- **Terminal** — live terminal output stream
- **Debug** — console errors, network failures, and system health in real-time

### Architecture

```
Terminal sessions          LocalPOV MCP Server          AI Agent
  (bash/zsh/ps)                   |                  (Claude Code,
       |                          |                   Cursor, etc.)
       |  ~/.localpov/sessions/   |   read_terminal         |
       |------------------------->|<------------------------|
                                  |                         |
Browser --> localpov proxy (:4000)|   read_browser          |
              |  injects capture  |<------------------------|
              |  script into HTML |                         |
              |                   |   take_screenshot       |
              |  console + fetch  |<------------------------|
              |  errors captured  |                         |
                                  |   get_diagnostics       |
Docker containers                 |<------------------------|
  |  docker logs                  |                         |
  |------------------------------>|   docker                |
                                  |<------------------------|
```

## Config file

Create `.localpovrc` in your project root:

```json
{
  "port": 3000,
  "listen": 4000,
  "ports": [3000, 3001, 5173, 8080]
}
```

## Supported shells

| Shell | Capture method | Platforms |
|-------|---------------|-----------|
| bash | `script` command + preexec/precmd hooks | Linux, macOS |
| zsh | `script` command + preexec/precmd hooks | Linux, macOS |
| fish | `script` command + fish_preexec/fish_postexec | Linux, macOS |
| PowerShell | `Start-Transcript` + prompt function flush | Windows, Linux, macOS |

## Safety

- **Output capped at 50KB** per MCP response — prevents flooding the AI context
- **Environment values never exposed** — `check_env` only reports existence (true/false)
- **File access restricted** — `tail_log` only reads from cwd, `~/.localpov/`, `/var/log/`, `/tmp/`
- **Sensitive paths blocked** — `.ssh`, `.env`, credentials files are never readable
- **WebSocket limits** — max 50 connections, 64KB message size cap
- **Stale data cleanup** — sessions >24h and browser data >4h auto-cleaned

## Requirements

- Node.js 18+

## License

MIT
