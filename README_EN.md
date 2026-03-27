# Feishu / Lark

Connect a Feishu (飞书) or Lark bot to your Claude Code session via an MCP server.

The MCP server connects to Feishu via WebSocket long-polling and provides tools to Claude for replying, reacting, and editing messages. When you message the bot, the server forwards the message to your Claude Code session.

## Prerequisites

- [Bun](https://bun.sh) — the MCP server runs on Bun. Install with `curl -fsSL https://bun.sh/install | bash`.
- A Feishu/Lark app with bot capability enabled.
- **Claude Code v2.1.80+**, and you must log in via **claude.ai** (`claude login`).
- **The Channel feature does not support API key authentication** (including `ANTHROPIC_API_KEY`, `ANTHROPIC_AUTH_TOKEN`, and third-party proxies like DashScope). When using API keys, the MCP server connects and receives messages normally, but Claude Code will not process `notifications/claude/channel` notifications.
- Claude **Max** plan, or Team/Enterprise plan (requires admin to enable `channelsEnabled`).

## Quick Setup

**1. Create a Feishu app.**

Go to the [Feishu Open Platform](https://open.feishu.cn) (or [Lark Developer](https://open.larksuite.com) for international):

1. Create a new app (自建应用)
2. Enable **Bot** capability (机器人)
3. Add permissions:
   - `im:message` — receive messages
   - `im:message:send_as_bot` — send messages
   - `im:message.reactions:write` — add reactions (optional)
4. Subscribe to events:
   - Add event `im.message.receive_v1` (receive messages)
   - Set subscription mode to **Receive events through persistent connection**
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
# After official release
claude --channels plugin:feishu@claude-plugins-official

# During development (local plugin)
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

## Access Control

See **[ACCESS.md](./ACCESS.md)** for DM policies, the `access.json` schema, and all `/feishu:setup` subcommands.

Quick reference: IDs are Feishu **open_id** values (like `ou_xxxx`). Default policy is `pairing`.

## Tools Exposed to the Assistant

| Tool | Purpose |
| --- | --- |
| `reply` | Send to a chat. Takes `chat_id` + `text`, optionally `reply_to` (message ID) for native threading. Auto-chunks long text. Returns the sent message ID(s). |
| `react` | Add an emoji reaction to a message by ID. Uses Feishu emoji type strings like `THUMBSUP`, `SMILE`, `HEART`. |
| `edit_message` | Edit a message the bot previously sent. Useful for "working..." -> result progress updates. |

## No History or Search

Feishu's Bot API via WebSocket exposes **neither** message history nor search. The bot only sees messages as they arrive. If the assistant needs earlier context, it will ask you to paste or summarize.

## Local Development

### Preparation

```bash
# Clone the repo and install dependencies
git clone https://github.com/your-org/claude-feishu-channel.git
cd claude-feishu-channel
bun install

# Write credentials (first time)
mkdir -p ~/.claude/channels/feishu
cat > ~/.claude/channels/feishu/.env << 'EOF'
FEISHU_APP_ID=cli_xxxx
FEISHU_APP_SECRET=your_app_secret_here
EOF
chmod 600 ~/.claude/channels/feishu/.env
```

### Option A: --plugin-dir (recommended, full plugin features)

Load as a plugin with `/feishu:setup` skill available:

```bash
claude --plugin-dir /path/to/claude-feishu-channel
```

`--plugin-dir` can be used multiple times to load multiple plugins. The flag is required on every launch.

You can add an alias in `~/.zshrc` or `~/.bashrc` for convenience:

```bash
alias claude-feishu='claude --plugin-dir /path/to/claude-feishu-channel'
```

### Option B: --dangerously-load-development-channels (bare MCP server)

If you don't need the plugin wrapper (skills, etc.), you can load it directly as an MCP server. This is the officially recommended debugging method during the research preview stage.

1. Register the server in `~/.claude.json`:

```json
{
  "mcpServers": {
    "feishu": {
      "command": "bun",
      "args": ["/path/to/claude-feishu-channel/server.ts"]
    }
  }
}
```

2. Start Claude Code:

```bash
claude --dangerously-load-development-channels server:feishu
```

> **Note**: The `/feishu:setup` skill is not available in this mode. You'll need to manage credentials and `access.json` manually. This flag only bypasses the allowlist check; the organization policy `channelsEnabled` still applies.

### Configuring Credentials (two methods)

**Option A**: Use the `/feishu:setup` skill (available after loading as a plugin)

```
/feishu:setup cli_xxxx your_app_secret_here
```

**Option B**: Manually edit `~/.claude/channels/feishu/.env`

```
FEISHU_APP_ID=cli_xxxx
FEISHU_APP_SECRET=your_app_secret_here
FEISHU_DOMAIN=feishu
```

> Restart the Claude Code session after changing credentials for the changes to take effect.

### Debugging

View MCP server log output (written to stderr):

```bash
# Run the server manually to test the connection
cd /path/to/claude-feishu-channel
bun server.ts
```

If the WebSocket connection succeeds, you'll see:

```
feishu channel: WebSocket connected (appId: cli_xxxx)
```

> **Team/Enterprise users**: If you see "blocked by org policy", ask your admin to enable `channelsEnabled` in [Admin Settings](https://claude.ai/admin-settings/claude-code).

### Troubleshooting

**Feishu messages sent, but Claude Code doesn't respond**

The Channel feature is in research preview with the following limitations:

1. **Must log in via claude.ai** (`claude login`). API key authentication (`ANTHROPIC_API_KEY`, `ANTHROPIC_AUTH_TOKEN`, third-party proxies, etc.) does not support channel inbound notifications. The MCP server runs normally and Feishu messages are received, but the Claude Code client silently ignores `notifications/claude/channel`.
2. **Requires Max plan** (or Team/Enterprise with admin-enabled `channelsEnabled`).
3. Custom channels during development are not on the official allowlist — make sure Claude Code version ≥ 2.1.80.

Troubleshooting steps:

```bash
# Check authentication method
claude auth status

# Check version
claude --version  # Must be >= 2.1.80
```

If you're using an API key or third-party proxy, switch to claude.ai login:

```bash
claude login
```

Reference: [Channels reference](https://code./claude.com/docs/en/channels-reference) — "Channels require claude.ai login. Console and API key authentication is not supported."

### After Official Release

Once the plugin is added to the official marketplace, users can install and launch with the standard method:

```bash
claude --channels plugin:feishu@claude-plugins-official
```

## Environment Variables

| Variable | Description |
| --- | --- |
| `FEISHU_APP_ID` | Feishu app ID (required) |
| `FEISHU_APP_SECRET` | Feishu app secret (required) |
| `FEISHU_DOMAIN` | `feishu` (default), `lark`, or custom URL |
| `FEISHU_STATE_DIR` | Override state directory (default: `~/.claude/channels/feishu`) |
