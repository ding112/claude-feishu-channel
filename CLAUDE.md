# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 项目概述

这是一个飞书/Lark 机器人的 MCP (Model Context Protocol) 服务器插件，用于将飞书机器人连接到 Claude Code 会话。

## 常用命令

```bash
# 安装依赖
bun install

# 启动 MCP 服务器（调试用）
bun server.ts

# 完整启动（安装依赖 + 启动）
bun start
```

## 架构

项目采用单文件架构，核心逻辑全部在 `server.ts` 中：

```
server.ts
├── 状态管理 (~/.claude/channels/feishu/)
│   ├── .env - 凭证 (FEISHU_APP_ID, FEISHU_APP_SECRET, FEISHU_DOMAIN)
│   └── access.json - 访问控制
├── 访问控制 (gate 函数)
│   └── 三种 DM 策略: pairing / allowlist / disabled
├── MCP 服务器
│   └── 三个工具: reply, react, edit_message
└── Feishu WebSocket 客户端
    └── 事件监听: im.message.receive_v1
```

### 消息流向

1. 用户在飞书发送消息给机器人
2. WebSocket 接收 `im.message.receive_v1` 事件
3. `gate()` 函数检查访问权限（deliver/drop/pair）
4. 通过后，消息通过 MCP notification 发送到 Claude Code
5. Claude 通过 MCP tools (reply/react/edit_message) 回复

### 关键文件

| 文件 | 用途 |
|------|------|
| `server.ts` | MCP 服务器主入口，包含所有业务逻辑 |
| `skills/setup/SKILL.md` | `/feishu:setup` 技能定义，管理凭证和访问控制 |
| `.mcp.json` | MCP 服务器启动配置 |
| `.claude-plugin/plugin.json` | 插件元数据 |

## 技术栈

- **运行时**: Bun
- **MCP SDK**: `@modelcontextprotocol/sdk`
- **飞书 SDK**: `@larksuiteoapi/node-sdk`
- **数据验证**: Zod

## 访问控制

- **pairing** (默认): 未知用户触发配对流程，返回 6 位配对码
- **allowlist**: 仅白名单用户可访问，其他静默丢弃
- **disabled**: 禁用所有消息

用户 ID 使用飞书 `open_id` 格式（如 `ou_xxx`）。

## 开发注意事项

- 修改凭证后需重启 Claude Code 会话
- `access.json` 修改立即生效（每条消息重新读取）
- 飞书 Bot API 不提供历史消息和搜索功能
- 消息分块限制 4000 字符