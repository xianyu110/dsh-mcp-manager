# dsh-mcp-manager

[简体中文](README.zh-CN.md) | English

**MCP server manager for [DeepSeek Harness (DSH)](https://github.com/deepseek-ai/deepseek-harness)** — a Settings → MCP page where you add MCP servers once, authenticate with **OAuth in the browser**, and get every server's tools registered as native `mcp__<name>__*` tools in all your sessions.

The built-in `@deepseek-ai/dsh-mcp-client` only accepts a static `headers` config — it has no OAuth support. This plugin fills that gap:

- **OAuth (authorization code + PKCE)** with RFC 7591 dynamic client registration, `refresh_token` rotation, and auto-reconnect across restarts — one browser login, then it keeps working.
- **Static Bearer token** mode for servers without OAuth.
- **Tool registration** with the same `mcp__<server>__<rawName>` naming convention as the built-in client, including strict-schema sanitization for the DSH tool registry and `isConcurrencySafe` marking.

## Requirements

- DeepSeek Harness with the `web` profile (`npx @deepseek-ai/dsh web`)
- Node.js `^22.19` or `>=24`; pnpm on your `PATH`

## Install

```sh
npx -p @deepseek-ai/dsh dsh plugin --profile web add github:hyqhyq3/dsh-mcp-manager
```

Then restart `dsh --profile web` and refresh the page. The package declares a `dsh.bundle.patch`, so the plugin activates automatically — no manual `cordis.patch.yml` editing.

> The MCP server's OAuth provider must allow a loopback redirect (`http://127.0.0.1:<port>/mcp-manager/callback/<id>`), which is where the DSH GUI webserver receives the code. The origin is derived from your browser's own address, so any host/port the GUI is served on works.

## Usage

1. Open **Settings → MCP** in the DSH web UI.
2. **＋ Add MCP server**: name (becomes the `mcp__<name>__*` prefix), URL, and auth mode (OAuth or static token).
3. OAuth servers: click **去认证 (Authenticate)** → the browser opens the server's login page → after consent you are redirected back and the tools are registered immediately.
4. Static-token servers connect as soon as the token is saved.

Status badges: `connected (N tools)` / `needs-auth` / `authorizing` / `error`. Buttons: authenticate, reconnect, delete. State persists at `~/.dsh/mcp-manager.json` (server configs + OAuth client registrations + tokens).

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
| Token storage | `~/.dsh/mcp-manager.json`; refreshed automatically on 401 |
| MCP transport | Streamable HTTP (JSON-RPC over POST, `Mcp-Session-Id`, SSE or JSON responses) |
| Tool schema | Server JSON Schemas are sanitized to the registry's supported raw subset (unsupported vocabulary degrades to unconstrained) |
| Hot path | Same-origin JSON API under `/mcp-manager/api/*` between the settings page and the host half |

## Limitations

- `resources` and `prompts` MCP capabilities are not bridged (tools only).
- Tokens live in a plain JSON file under `~/.dsh` — treat the file as a secret.
- One OAuth client registration per server per GUI origin; moving the GUI to a new origin re-registers automatically on the next login.

## License

MIT
