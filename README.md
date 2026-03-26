# Feishu / Lark

Connect a Feishu (飞书) or Lark bot to your Claude Code session via an MCP server.

The MCP server connects to Feishu via WebSocket long-polling and provides tools to Claude for replying, reacting, and editing messages. When you message the bot, the server forwards the message to your Claude Code session.

## Prerequisites

- [Bun](https://bun.sh) — the MCP server runs on Bun. Install with `curl -fsSL https://bun.sh/install | bash`.
- A Feishu/Lark app with bot capability enabled.

## Quick Setup

**1. Create a Feishu app.**

Go to the [Feishu Open Platform](https://open.feishu.cn) (or [Lark Developer](https://open.larksuite.com) for international):

1. Create a new app (自建应用)
2. Enable **Bot** capability (机器人)
3. Add permissions:
   - `im:message` — receive messages
   - `im:message:send_as_bot` — send messages
   - `im:message.reactions:write` — add reactions (optional)
4. Publish the app version and get admin approval
5. Copy the **App ID** (`cli_xxxx`) and **App Secret**

**2. Install the plugin.**

These are Claude Code commands — run `claude` to start a session first.

```
/plugin install feishu@claude-plugins-official
```

**3. Save your credentials.**

```
/feishu:setup cli_xxxx your_app_secret_here
```

Writes `FEISHU_APP_ID` and `FEISHU_APP_SECRET` to `~/.claude/channels/feishu/.env`.

For Lark (international) users, add the domain:

```
/feishu:setup cli_xxxx your_app_secret_here lark
```

**4. Relaunch with the channel flag.**

Exit your session and start a new one:

```sh
claude --channels plugin:feishu@claude-plugins-official
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

## Environment variables

| Variable | Description |
| --- | --- |
| `FEISHU_APP_ID` | Feishu app ID (required) |
| `FEISHU_APP_SECRET` | Feishu app secret (required) |
| `FEISHU_DOMAIN` | `feishu` (default), `lark`, or custom URL |
| `FEISHU_STATE_DIR` | Override state directory (default: `~/.claude/channels/feishu`) |
