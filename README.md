# 飞书 / Lark

[English](./README_EN.md)

通过 MCP server 将飞书（Feishu）或 Lark 机器人连接到你的 Claude Code 会话。

MCP server 通过 WebSocket 长连接接入飞书，为 Claude 提供回复、表情回应和编辑消息的工具。当你给机器人发消息时，server 会将消息转发到你的 Claude Code 会话。

## 前置条件

- [Bun](https://bun.sh) — MCP server 基于 Bun 运行。安装命令：`curl -fsSL https://bun.sh/install | bash`。
- 一个已启用机器人能力的飞书/Lark 应用。
- **Claude Code v2.1.80+**，且必须通过 **claude.ai 登录**（`claude login`）。
- **Channel 功能不支持 API key 认证**（包括 `ANTHROPIC_API_KEY`、`ANTHROPIC_AUTH_TOKEN` 及第三方代理如 DashScope）。使用 API key 时，MCP server 能正常连接和接收消息，但 Claude Code 不会处理 `notifications/claude/channel` 通知。
- Claude **Max** 计划，或 Team/Enterprise 计划（需管理员启用 `channelsEnabled`）。

## 快速开始

**1. 创建飞书应用**

前往[飞书开放平台](https://open.feishu.cn)（国际版请访问 [Lark Developer](https://open.larksuite.com)）：

1. 创建一个自建应用
2. 启用**机器人**能力
3. 添加权限：
   - `im:message` — 接收消息
   - `im:message:send_as_bot` — 以机器人身份发送消息
   - `im:message.reactions:write` — 添加表情回应（可选）
4. 订阅事件（事件与回调）：
   - 添加事件 `im.message.receive_v1`（接收消息）
   - 订阅方式选择**使用长连接接收事件**
5. 发布应用版本并获取管理员审批
6. 复制 **App ID**（`cli_xxxx`）和 **App Secret**

**2. 安装插件**

以下是 Claude Code 命令 — 先运行 `claude` 启动一个会话。

正式发布到官方市场后：

```
/plugin install feishu@claude-plugins-official
```

本地开发请参阅下方[本地开发](#本地开发)。

**3. 保存凭证**

```
/feishu:setup cli_xxxx your_app_secret_here
```

会将 `FEISHU_APP_ID` 和 `FEISHU_APP_SECRET` 写入 `~/.claude/channels/feishu/.env`。

Lark（国际版）用户需要指定域名：

```
/feishu:setup cli_xxxx your_app_secret_here lark
```

**4. 重新启动并加载插件**

退出当前会话，启动新会话：

```sh
# 正式发布后
claude --channels plugin:feishu@claude-plugins-official

# 开发阶段（本地插件）
claude --plugin-dir /path/to/claude-feishu-channel
```

**5. 配对**

Claude Code 运行后，在飞书上给机器人发私信 — 机器人会回复一个 6 位配对码。在 Claude Code 会话中输入：

```
/feishu:setup pair <code>
```

之后你的私信就能直达助手了。

**6. 锁定访问**

配对成功后，切换为白名单模式，防止陌生人获取配对码：

```
/feishu:setup policy allowlist
```

## 访问控制

详见 **[ACCESS.md](./ACCESS.md)**，了解私信策略、`access.json` 格式以及所有 `/feishu:setup` 子命令。

快速参考：ID 为飞书 **open_id** 值（如 `ou_xxxx`）。默认策略为 `pairing`。

## 助手可用的工具

| 工具 | 用途 |
| --- | --- |
| `reply` | 向聊天发送消息。接收 `chat_id` + `text` 参数，可选 `reply_to`（消息 ID）用于原生话题回复。自动分段发送长文本。返回已发送的消息 ID。 |
| `react` | 根据消息 ID 添加表情回应。使用飞书表情类型字符串，如 `THUMBSUP`、`SMILE`、`HEART`。 |
| `edit_message` | 编辑机器人之前发送的消息。适用于"处理中..." → 结果的进度更新场景。 |

## 无历史记录和搜索功能

飞书 Bot API 通过 WebSocket **不提供**消息历史记录和搜索功能。机器人只能看到实时到达的消息。如果助手需要之前的上下文，会请你粘贴或概述。

## 本地开发

### 准备工作

```bash
# 克隆仓库并安装依赖
git clone https://github.com/your-org/claude-feishu-channel.git
cd claude-feishu-channel
bun install

# 写入凭证（首次）
mkdir -p ~/.claude/channels/feishu
cat > ~/.claude/channels/feishu/.env << 'EOF'
FEISHU_APP_ID=cli_xxxx
FEISHU_APP_SECRET=your_app_secret_here
EOF
chmod 600 ~/.claude/channels/feishu/.env
```

### 方式 A：--plugin-dir（推荐，完整插件功能）

以插件形式加载，`/feishu:setup` skill 可用：

```bash
claude --plugin-dir /path/to/claude-feishu-channel
```

`--plugin-dir` 支持重复使用以加载多个插件。每次启动需要带此参数。

可以在 `~/.zshrc` 或 `~/.bashrc` 中添加 alias 简化：

```bash
alias claude-feishu='claude --plugin-dir /path/to/claude-feishu-channel'
```

### 方式 B：--dangerously-load-development-channels（裸 MCP server）

如果不需要插件包装（skill 等功能），可以直接以 MCP server 方式加载。这是官方文档推荐的 research preview 阶段调试方式。

1. 在 `~/.claude.json` 中注册 server：

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

2. 启动 Claude Code：

```bash
claude --dangerously-load-development-channels server:feishu
```

> **注意**：此方式下 `/feishu:setup` skill 不可用，需手动管理凭证和 `access.json`。该标志只跳过白名单检查，组织策略 `channelsEnabled` 仍然生效。

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

## 环境变量

| 变量 | 说明 |
| --- | --- |
| `FEISHU_APP_ID` | 飞书应用 ID（必填） |
| `FEISHU_APP_SECRET` | 飞书应用密钥（必填） |
| `FEISHU_DOMAIN` | `feishu`（默认）、`lark` 或自定义 URL |
| `FEISHU_STATE_DIR` | 覆盖状态目录（默认：`~/.claude/channels/feishu`） |
