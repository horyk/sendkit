---
name: sendkit
description: Send Telegram messages through the SendKit MCP server (mcp__sendkit__telegram), and diagnose the server when it fails to connect. Use this whenever the user asks to send, notify, or ping someone via SendKit or Telegram, or gives a numeric chat ID with a request like "send a message to 123456789", "notify me on Telegram", "〇〇に送って", or "通知して", even if they don't say "MCP" or "Telegram" explicitly. Also use when the sendkit MCP server shows CONNECTION_CLOSED or its tools are missing.
---

# SendKit

SendKit sends messages from an AI agent to messaging apps. The MCP server `@horykoji/sendkit-mcp` exposes a `telegram` tool, which takes a `chatId` and a `message` and sends the message through the user's Telegram bot.

## Sending a message

1. **Load the tool.** Some clients load MCP tools lazily. If the `mcp__sendkit__telegram` schema isn't loaded yet, load it first. In Claude Code, call `ToolSearch` with `select:mcp__sendkit__telegram`.
2. **Work out the arguments.**
   - `chatId`: pass it as a string, exactly as the user gave it. Group chat IDs are negative (e.g. `-100…`), so keep the sign.
   - `message`: if the user wrote the text, send it unchanged. If they left it open ("send something", "何かメッセージ"), write one short, friendly test message in the user's language that says it came from SendKit. Don't add anything private, alarming, or that could pass for a real announcement.
   - If the recipient is unclear (no chat ID, or several candidates), ask before sending. A sent message can't be taken back.
3. **Send it once.** Call `mcp__sendkit__telegram`. On success the result looks like `{"ok":true,"chatId":"…","messageId":12}`.
4. **Report back** in a line or two, in the user's language: who it went to, the text, and the `messageId`. If messages already went to the same chat earlier in the conversation, say so, so the user knows how many arrived.

When the user asks to send again, go ahead. Don't retry silently after an error, though, because a timeout can still mean the message got delivered.

## When the MCP server is unavailable

The usual symptoms: the client reports `sendkit` as failed or `CONNECTION_CLOSED`, or no `mcp__sendkit__*` tools exist.

The user chose to send through the MCP server, so don't quietly switch to another route. Diagnose the server instead:

1. **Find the server config.** In Claude Code that's `.mcp.json` in the project, or the user-level config. Other clients keep their own MCP config file. The `sendkit` entry should look like this:
   ```json
   "sendkit": {
     "command": "npx",
     "args": ["-y", "@horykoji/sendkit-mcp"],
     "env": { "TELEGRAM_BOT_TOKEN": "<your bot token>" }
   }
   ```
   Common mistakes:
   - `"args": ["run", "@horykoji/sendkit-mcp"]`: npx reads `run` as the package name, so the server never starts. Use `-y` instead.
   - `TELEGRAM_BOT_TOKEN` missing or empty. The token comes from Telegram's @BotFather.
   - Node.js/npx isn't installed or isn't on the client's `PATH`.
2. **Fix what's clearly wrong** in the config, then ask the user to reconnect the server. In Claude Code that's `/mcp`; other clients may need a restart. You can't reconnect it yourself, and it can take a couple of attempts.
3. **Send through the MCP** once it's back.

If the user wants the message out before the server is fixed, offer the SendKit CLI as an explicit option, and use it only if they agree:

```bash
npx -y @horykoji/sendkit init --telegram-bot-token <token>   # one-time setup, stores the token in ~/.sendkit/config.json
npx -y @horykoji/sendkit telegram <chatId> "<message>"
```

## Handling the token

The bot token often sits in plain text in an MCP config file. Never echo it in output or paste it into messages. If that config file is tracked by version control, point out once that the token should move to an environment variable or an untracked file.
