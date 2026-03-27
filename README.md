# Feishu / Lark

Connect a Feishu (飞书) or Lark bot to your Claude Code session via an MCP server.

The MCP server connects to Feishu via WebSocket long-polling and provides tools to Claude for replying, reacting, and editing messages. When you message the bot, the server forwards the message to your Claude Code session.

## Prerequisites

- [Bun](https://bun.sh) — the MCP server runs on Bun. Install with `curl -fsSL https://bun.sh/install | bash`.
- A Feishu/Lark app with bot capability enabled.
- **Claude Code v2.1.80+**，且必须通过 **claude.ai 登录**（`claude login`）。
- **Channel 功能不支持 API key 认证**（包括 `ANTHROPIC_API_KEY`、`ANTHROPIC_AUTH_TOKEN` 及第三方代理如 DashScope）。使用 API key 时，MCP server 能正常连接和接收消息，但 Claude Code 不会处理 `notifications/claude/channel` 通知。
- Claude **Max** 计划，或 Team/Enterprise 计划（需管理员启用 `channelsEnabled`）。

## Quick Setup

**1. Create a Feishu app.**

Go to the [Feishu Open Platform](https://open.feishu.cn) (or [Lark Developer](https://open.larksuite.com) for international):

1. Create a new app (自建应用)
2. Enable **Bot** capability (机器人)
3. Add permissions:
   - `im:message` — receive messages
   - `im:message:send_as_bot` — send messages
   - `im:message.reactions:write` — add reactions (optional)
4. Subscribe to events (事件与回调):
   - Add event `im.message.receive_v1` (接收消息)
   - Set subscription mode to **Receive events through persistent connection** (使用长连接接收事件)
5. Publish the app version and get admin approval
6. Copy the **App ID** (`cli_xxxx`) and **App Secret**

**2. Install the plugin.**

These are Claude Code commands — run `claude` to start a session first.

Once published to the official marketplace:

```
/plugin install feishu@claude-plugins-official
```

For local development, see [Local Development](#local-development) below.

**3. Save your credentials.**

```
/feishu:setup cli_xxxx your_app_secret_here
```

Writes `FEISHU_APP_ID` and `FEISHU_APP_SECRET` to `~/.claude/channels/feishu/.env`.

For Lark (international) users, add the domain:

```
/feishu:setup cli_xxxx your_app_secret_here lark
```

**4. Relaunch with the plugin loaded.**

Exit your session and start a new one:

```sh
# 正式发布后
claude --channels plugin:feishu@claude-plugins-official

# 开发阶段（本地插件）
claude --plugin-dir /path/to/claude-feishu-channel
```

**5. Pair.**

With Claude Code running, DM your bot on Feishu — it replies with a 6-character pairing code. In your Claude Code session:

```
/feishu:setup pair <code>
```

Your next DM reaches the assistant.

**6. Lock it down.**

Once you're paired, switch to allowlist so strangers don't get pairing-code replies:

```
/feishu:setup policy allowlist
```

## Access control

See **[ACCESS.md](./ACCESS.md)** for DM policies, the `access.json` schema, and all `/feishu:setup` subcommands.

Quick reference: IDs are Feishu **open_id** values (like `ou_xxxx`). Default policy is `pairing`.

## Tools exposed to the assistant

| Tool | Purpose |
| --- | --- |
| `reply` | Send to a chat. Takes `chat_id` + `text`, optionally `reply_to` (message ID) for native threading. Auto-chunks long text. Returns the sent message ID(s). |
| `react` | Add an emoji reaction to a message by ID. Uses Feishu emoji type strings like `THUMBSUP`, `SMILE`, `HEART`. |
| `edit_message` | Edit a message the bot previously sent. Useful for "working..." -> result progress updates. |

## No history or search

Feishu's Bot API via WebSocket exposes **neither** message history nor search. The bot only sees messages as they arrive. If the assistant needs earlier context, it will ask you to paste or summarize.

## Local Development

### 加载本地插件

```bash
# 1. 克隆仓库并安装依赖
git clone https://github.com/your-org/claude-feishu-channel.git
cd claude-feishu-channel
bun install

# 2. 手动写入凭证（首次）
mkdir -p ~/.claude/channels/feishu
cat > ~/.claude/channels/feishu/.env << 'EOF'
FEISHU_APP_ID=cli_xxxx
FEISHU_APP_SECRET=your_app_secret_here
EOF
chmod 600 ~/.claude/channels/feishu/.env

# 3. 启动 Claude Code（--plugin-dir 加载本地插件）
claude --plugin-dir /path/to/claude-feishu-channel
```

`--plugin-dir` 支持重复使用以加载多个插件。每次启动需要带此参数。

### 简化启动（shell alias）

每次输入完整命令较繁琐，可以在 `~/.zshrc` 或 `~/.bashrc` 中添加 alias：

```bash
alias claude-feishu='claude --plugin-dir /path/to/claude-feishu-channel'
```

之后只需运行 `claude-feishu` 即可。

### 配置凭证（两种方式）

**方式 A**：用 `/feishu:setup` skill（插件加载后可用）

```
/feishu:setup cli_xxxx your_app_secret_here
```

**方式 B**：手动编辑 `~/.claude/channels/feishu/.env`

```
FEISHU_APP_ID=cli_xxxx
FEISHU_APP_SECRET=your_app_secret_here
FEISHU_DOMAIN=feishu
```

> 修改凭证后需要重启 Claude Code 会话才能生效。

### 调试

查看 MCP server 日志输出（写入 stderr）：

```bash
# 手动运行 server 测试连接是否正常
cd /path/to/claude-feishu-channel
bun server.ts
```

如果 WebSocket 连接成功，会输出：

```
feishu channel: WebSocket connected (appId: cli_xxxx)
```

> **Team/Enterprise 用户**：如果看到 "blocked by org policy"，需要管理员在 [Admin Settings](https://claude.ai/admin-settings/claude-code) 中启用 `channelsEnabled`。

### 常见问题

**飞书消息已发送，但 Claude Code 没有反应**

Channel 功能处于 research preview 阶段，有以下限制：

1. **必须使用 claude.ai 登录**（`claude login`）。API key 认证（`ANTHROPIC_API_KEY`、`ANTHROPIC_AUTH_TOKEN`、第三方代理等）不支持 channel inbound 通知。此时 MCP server 正常运行、飞书消息正常接收，但 Claude Code 客户端会静默忽略 `notifications/claude/channel`。
2. **需要 Max 计划**（或 Team/Enterprise 且管理员已启用 `channelsEnabled`）。
3. 开发阶段的自定义 channel 不在官方白名单中，需要确认 Claude Code 版本 ≥ 2.1.80。

排查步骤：

```bash
# 确认认证方式
claude auth status

# 确认版本
claude --version  # 需要 >= 2.1.80
```

如果你使用了 API key 或第三方代理，需要切换为 claude.ai 登录：

```bash
claude login
```

参考：[Channels reference](https://code./claude.com/docs/en/channels-reference) — "Channels require claude.ai login. Console and API key authentication is not supported."

### 正式发布后

一旦插件被加入官方 marketplace，用户就可以用标准方式安装和启动：

```bash
claude --channels plugin:feishu@claude-plugins-official
```

## Environment variables

| Variable | Description |
| --- | --- |
| `FEISHU_APP_ID` | Feishu app ID (required) |
| `FEISHU_APP_SECRET` | Feishu app secret (required) |
| `FEISHU_DOMAIN` | `feishu` (default), `lark`, or custom URL |
| `FEISHU_STATE_DIR` | Override state directory (default: `~/.claude/channels/feishu`) |
