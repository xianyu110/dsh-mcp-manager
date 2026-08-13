# dsh-mcp-manager

[English](README.md) | 简体中文

**[DeepSeek Harness (DSH)](https://github.com/deepseek-ai/deepseek-harness) 的 MCP 服务器管理插件** —— 在 设置 → MCP 页签里添加 MCP 服务器，**浏览器里完成 OAuth 登录**，服务器的全部工具即注册为所有会话可用的原生 `mcp__<name>__*` 工具。

内置的 `@deepseek-ai/dsh-mcp-client` 只接受静态 `headers` 配置——不支持 OAuth。本插件补上这块：

- **OAuth（授权码 + PKCE）**：RFC 7591 动态客户端注册、`refresh_token` 自动轮换、重启后自动重连——浏览器登录一次，之后一直可用。
- **静态 Bearer Token** 模式：适配没有 OAuth 的服务器。
- **工具注册**：与内置客户端相同的 `mcp__<server>__<rawName>` 命名约定，含 DSH 工具注册表的严格 schema 清洗，并标记 `isConcurrencySafe`。

## 前置要求

- DeepSeek Harness web profile（`npx @deepseek-ai/dsh web`）
- Node.js `^22.19` 或 `>=24`；`PATH` 里有 pnpm

## 安装

```sh
npx -p @deepseek-ai/dsh dsh plugin --profile web add github:hyqhyq3/dsh-mcp-manager
```

重启 `dsh --profile web` 并刷新页面。包内声明了 `dsh.bundle.patch`，插件自动激活——无需手动改 `cordis.patch.yml`。

> MCP 服务器的 OAuth 提供方需要允许回环重定向（`http://127.0.0.1:<port>/mcp-manager/callback/<id>`）——DSH GUI 自身的 webserver 会接收授权码。origin 从浏览器实际地址动态派生，GUI 用任意 host/port 访问都可以。

## 使用

1. 打开 DSH Web UI 的 **设置 → MCP**。
2. **＋ 添加 MCP 服务器**：名称（决定 `mcp__<name>__*` 前缀）、URL、认证方式（OAuth 或静态 token）。
3. OAuth 服务器：点 **去认证** → 浏览器打开登录页 → 同意授权后跳回，工具立即注册。
4. 静态 token 服务器：保存 token 后即连接。

状态徽章：`已连接 (N 个工具)` / `待认证` / `认证中` / `错误`。按钮：去认证、重连、删除。状态持久化在 `~/.dsh/mcp-manager.json`（服务器配置 + OAuth 客户端注册信息 + token）。

### Agent 看到什么

每个已连接服务器的工具以一等工具出现，例如名为 `odin` 的服务器：

```
mcp__odin__search_tools     mcp__odin__describe_tool
mcp__odin__execute_tool     mcp__odin__list_tool_scopes
```

工具结果渲染为原生文本内容；`isError` 结果走注册表错误路径。

## 工作原理

| 组成 | 机制 |
|---|---|
| 设置页 | client 半注册 `settings.section` 槽位（MCP 页签） |
| OAuth 流程 | host 半做动态客户端注册 + PKCE；重定向落在 DSH GUI webserver 自身挂载的路由上 |
| Token 存储 | `~/.dsh/mcp-manager.json`；401 时自动刷新 |
| MCP 传输 | Streamable HTTP（POST JSON-RPC、`Mcp-Session-Id`、SSE/JSON 双格式响应） |
| 工具 schema | 服务器 JSON Schema 清洗为注册表支持的 raw 子集（不支持的关键字降级为无约束） |
| 交互通道 | 设置页与 host 半之间走同源 JSON API（`/mcp-manager/api/*`） |

## 已知限制

- 只桥接 MCP 的工具能力（resources / prompts 不支持）。
- token 明文存于 `~/.dsh` 下的 JSON 文件——请当作机密对待。
- 每个 GUI origin 一次 OAuth 客户端注册；GUI 换地址后下次登录会自动重新注册。

## License

MIT
