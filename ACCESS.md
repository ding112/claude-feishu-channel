# Feishu — Access Control

A Feishu bot is addressable by anyone in the organization. Without a gate, messages from any user would flow into your assistant session. The access model described here decides who gets through.

By default, a DM from an unknown sender triggers **pairing**: the bot replies with a 6-character code and drops the message. You run `/feishu:setup pair <code>` from your assistant session to approve them. Once approved, their messages pass through.

All state lives in `~/.claude/channels/feishu/access.json`. The `/feishu:setup` skill commands edit this file; the server re-reads it on every inbound message, so changes take effect without a restart.

## At a glance

| | |
| --- | --- |
| Default policy | `pairing` |
| Sender ID | Feishu open_id (e.g. `ou_xxxxxxxxxxxx`) |
| Config file | `~/.claude/channels/feishu/access.json` |

## DM policies

`dmPolicy` controls how DMs from senders not on the allowlist are handled.

| Policy | Behavior |
| --- | --- |
| `pairing` (default) | Reply with a pairing code, drop the message. Approve with `/feishu:setup pair <code>`. |
| `allowlist` | Drop silently. No reply. |
| `disabled` | Drop everything, including allowlisted users. |

```
/feishu:setup policy allowlist
```

## User IDs

Feishu identifies users by **open_id** values like `ou_xxxxxxxxxxxx`. These are app-scoped — the same person has different open_ids across different apps.

Pairing captures the ID automatically. To find one manually, check the Feishu admin console or use the Feishu API.

```
/feishu:setup allow ou_xxxxxxxxxxxx
/feishu:setup remove ou_xxxxxxxxxxxx
```

## Group messages

In the current minimal implementation, group messages are delivered only if the sender's open_id is in `allowFrom`. Full group policy support (per-group config, mention detection) may be added in a future version.

## Skill reference

| Command | Effect |
| --- | --- |
| `/feishu:setup` | Print current state: credentials, policy, allowlist, pending pairings. |
| `/feishu:setup <appId> <appSecret>` | Save Feishu app credentials. |
| `/feishu:setup <appId> <appSecret> <domain>` | Save credentials with domain (`feishu`, `lark`, or custom URL). |
| `/feishu:setup clear` | Remove saved credentials. |
| `/feishu:setup pair a4f91c` | Approve pairing code. Adds the sender to `allowFrom`. |
| `/feishu:setup deny a4f91c` | Discard a pending code. The sender is not notified. |
| `/feishu:setup allow ou_xxx` | Add a user ID directly. |
| `/feishu:setup remove ou_xxx` | Remove from the allowlist. |
| `/feishu:setup policy allowlist` | Set `dmPolicy`. Values: `pairing`, `allowlist`, `disabled`. |

## Config file

`~/.claude/channels/feishu/access.json`. Absent file is equivalent to `pairing` policy with empty lists.

```jsonc
{
  // Handling for DMs from senders not in allowFrom.
  "dmPolicy": "pairing",

  // Feishu open_id values allowed to DM.
  "allowFrom": ["ou_xxxxxxxxxxxx"],

  // Active pairing requests (managed by the server, approved via /feishu:setup).
  "pending": {}
}
```
