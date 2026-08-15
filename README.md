# dsh-mcp-manager

[简体中文](README.zh-CN.md) | English

**MCP server manager for [DeepSeek Harness (DSH)](https://github.com/deepseek-ai/deepseek-harness)** — a Settings → MCP page where you add MCP servers once (remote HTTP or local stdio process), authenticate HTTP servers with **OAuth in the browser**, and get every server's tools registered as native `mcp__<name>__*` tools in all your sessions.

The built-in `@deepseek-ai/dsh-mcp-client` only accepts a static `headers` config — it has no OAuth support and no local stdio transport. This plugin fills that gap:

- **OAuth (authorization code + PKCE)** with RFC 7591 dynamic client registration, `refresh_token` rotation, and auto-reconnect across restarts — one browser login, then it keeps working.
- **Static Bearer token** mode for servers without OAuth — stored as an environment-variable **name** (Codex-style `tokenEnv`), never as plaintext in the config.
- **Custom HTTP headers** (`headers` for direct values, `headerEnv` for values read from environment variables) — matches Codex's `http_headers` / `env_http_headers`.
- **stdio local processes**: run `npx` / `uvx` / `python` etc. directly; the plugin speaks JSON-RPC over the child's stdin/stdout (spawns the process, reconnects, and reaps it on exit) — no remote server or auth required. Windows `.cmd` shims (e.g. `npx.cmd`) are resolved through `cmd.exe`.
- **Edit-in-place**: rename a server, switch stdio ↔ HTTP, or change auth/headers without deleting and re-adding it.
- **Tool registration** with the same `mcp__<server>__<rawName>` naming convention as the built-in client, including strict-schema sanitization for the DSH tool registry and `isConcurrencySafe` marking.

## Requirements

- DeepSeek Harness with the `web` profile (`npx @deepseek-ai/dsh web`)
- Node.js `^22.19` or `>=24`; pnpm on your `PATH`
- Windows 10/11: stdio commands are launched via `cmd.exe` so `.cmd` shims (`npx`, `uvx`) resolve correctly

## Install

```sh
npx -p @deepseek-ai/dsh dsh plugin --profile web add github:hyqhyq3/dsh-mcp-manager
```

Then restart `dsh --profile web` and refresh the page. The package declares a `dsh.bundle.patch`, so the plugin activates automatically — no manual `cordis.patch.yml` editing.

> The MCP server's OAuth provider must allow a loopback redirect (`http://127.0.0.1:<port>/mcp-manager/callback/<id>`), which is where the DSH GUI webserver receives the code. The origin is derived from your browser's own address, so any host/port the GUI is served on works.

## Usage

1. Open **Settings → MCP** in the DSH web UI.
2. **＋ Add MCP server** (and later **编辑 / Edit** to change it):
   - **HTTP**: name (becomes the `mcp__<name>__*` prefix), URL, auth mode (OAuth or static token), and optional headers (`headers` direct values, `headerEnv` values read from env vars).
   - **stdio**: name, command (e.g. `npx`), args (one per row), env vars (key/value rows), and optional working directory.
3. OAuth servers: click **去认证 (Authenticate)** → the browser opens the server's login page → after consent you are redirected back and the tools are registered immediately.
4. Static-token servers: enter the **name of an environment variable** that holds the token (e.g. `MCP_BEARER_TOKEN`) — the token itself is never written to disk; stdio servers spawn and connect immediately on save.

Status badges: `connected (N tools)` / `needs-auth` / `authorizing` / `error` / `disabled`. Buttons: authenticate, edit, enable/disable (switch), delete. **Disable** unregisters that server's tools and drops its connection (config and OAuth tokens persist); **Enable** reconnects without re-authenticating. Disabled servers stay dormant across restarts. The toggle is global: it affects every session in this profile. State persists at `~/.dsh/mcp-manager.json` (server configs + OAuth client registrations + tokens; static tokens are referenced by env-var name, not stored).

### What the agent sees

Every connected server's tools appear as first-class tools, e.g. for a server named `odin`:

```
mcp__odin__search_tools     mcp__odin__describe_tool
mcp__odin__execute_tool     mcp__odin__list_tool_scopes
```

Tool results are rendered as native text content; `isError` results surface through the registry's error path.

## How it works

| Piece | Mechanism |
|---|---|
| Settings page | Client half registers a `settings.section` slot entry (MCP tab) |
| OAuth flow | Host half does dynamic client registration + PKCE; the redirect lands on a route mounted on the DSH GUI webserver itself |
| Token storage | `~/.dsh/mcp-manager.json`; OAuth tokens refreshed automatically on 401. Static tokens are read from the environment variable named by `tokenEnv` — never persisted |
| MCP transport (HTTP) | Streamable HTTP (JSON-RPC over POST, `Mcp-Session-Id`, SSE or JSON responses); custom `headers`/`headerEnv` merged into every request |
| MCP transport (stdio) | `child_process.spawn` a local command, JSON-RPC over stdin/stdout (newline-delimited); reconnect reaps the old process first. On Windows it spawns through `cmd.exe` so `.cmd` shims resolve |
| Tool schema | Server JSON Schemas are sanitized to the registry's supported raw subset (unsupported vocabulary degrades to unconstrained) |
| Hot path | Same-origin JSON API under `/mcp-manager/api/*` between the settings page and the host half |

## Limitations

- `resources` and `prompts` MCP capabilities are not bridged (tools only).
- OAuth tokens live in a plain JSON file under `~/.dsh` — treat the file as a secret. Static bearer tokens and `headerEnv` values are read from environment variables and never persisted.
- stdio servers run as long-lived child processes tied to the plugin lifecycle. On POSIX `args` are whitespace-tokenized (quotes protect args with spaces) with no shell expansion; on Windows the command line is passed to `cmd.exe`, so shell metacharacters (`&`, `|`, `>`, `%VAR%`) are interpreted — prefer absolute paths and quote args containing spaces there.
- One OAuth client registration per server per GUI origin; moving the GUI to a new origin re-registers automatically on the next login.

## License

MIT
