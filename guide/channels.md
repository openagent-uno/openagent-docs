# Channels

All channels support text, images, files, voice, and video. The "is writing" indicator shows the agent is working, and — in [live mode](#live-messages) (on by default) — each tool call and each span of the answer is posted as its own chat message as the turn unfolds.

All channels share the same command vocabulary: `/new`, `/stop`, `/status`, `/queue`, `/help`, `/usage`.

## Telegram

```yaml
channels:
  telegram:
    token: ${TELEGRAM_BOT_TOKEN}
    allowed_users: ["123456789", "987654321"]
```

Features: inline stop button, voice transcription (faster-whisper local or OpenAI Whisper API), markdown rendering via HTML parse mode.

## Discord

```yaml
channels:
  discord:
    token: ${DISCORD_BOT_TOKEN}
    allowed_users:
      - "123456789012345678"
    # allowed_guilds: []         # [] = any server
    # listen_channels: []        # [] = mention required
    # dm_only: false             # true = DMs only
```

`allowed_users` is **mandatory** — the channel refuses to start without it. Unauthorized messages are silently ignored. Supports native slash commands and an inline stop button.

## WhatsApp (Green API)

```yaml
channels:
  whatsapp:
    green_api_id: ${GREEN_API_ID}
    green_api_token: ${GREEN_API_TOKEN}
    allowed_users:
      - "391234567890"
```

No message editing and no inline buttons — users cancel with `/stop` text command.

## WebSocket (Desktop/Web App)

The desktop app and web app connect to the gateway over Iroh QUIC, authenticated by device certificates. No port configuration or shared tokens needed — the invite ticket carries the coordinator's NodeId, and the client dials directly via Iroh.

For development or custom clients, the loopback proxy exposes a local TCP endpoint:

```bash
# The loopback proxy bridges localhost to the agent over Iroh
openagent-cli proxy
```

This exposes `localhost:PORT` that acts as a plain HTTP/WS gateway, with the proxy handling Iroh transport and device cert presentation transparently.

## Running Multiple Channels

```bash
openagent serve                  # all configured channels
openagent serve -ch telegram     # specific channel only
```

## Live messages

By default every channel narrates a turn in the chat itself, the way Hermes does:

- the platform's **native "is writing" indicator** turns on where one exists (Telegram's typing dot, Discord's "Bot is typing…"); platforms without one (Slack, WhatsApp) simply lean on the step messages. There is no `Thinking…` placeholder message — the server reports reasoning as a boolean flag, never a chat bubble,
- each tool the agent uses is posted as its own message with a **friendly, human label** rather than the raw tool name — memory-vault work reads `🧠 Memorizing — <note>`, `📖 Recalling — <note>`, `🔍 Searching memory — "<query>"`, `🗑️ Forgetting — <note>`; a shell call reads `🔧 Running command`, a web search `🌐 Searching the web`, and a failure `⚠️ <label> failed: …`. The labels match the app's tool chips,
- each span of the assistant's text is posted as it is produced — the narration before a tool call, then the final answer — instead of one reply at the end.

The final message is never a duplicate: only the still-unposted tail of the answer is sent. This is one agent with many doorways (vision §9) — the same outbound stream the desktop app renders as a live transcript, mapped onto platform-native messages.

Turn it off per channel to get a single final reply per turn:

```yaml
channels:
  telegram:
    token: ${TELEGRAM_BOT_TOKEN}
    live: false        # one reply per turn; "is writing" indicator still shows
```

Or disable it fleet-wide with the `OPENAGENT_CHANNEL_LIVE=0` environment variable (a per-channel `live:` value still wins). Voice-note turns always use the single-reply path so a spoken question gets a spoken answer rather than a wall of intermediate text.

## Media Support

### Files & images from user → agent

Upload endpoint:

```
POST /api/upload        (multipart/form-data, field: "file")
```

Returns `{path, filename, transcription?}`. The `path` is a local absolute path the agent can read with the filesystem MCP (`filesystem_read_text_file`, `filesystem_read_media_file`, `filesystem_get_file_info`, etc.). On macOS it's a `/private/var/folders/.../T/oa_upload_<rand>/<filename>` realpath — already resolved so the filesystem MCP's allowlist check doesn't reject it.

Flow:

1. Client (web app, desktop, any bridge) posts the file to `/api/upload`.
2. The returned path goes into the next chat message text (e.g. `"Summarise the file at /private/var/.../report.pdf"`) — OR into the WS `attachments` field if the client builds one directly via `Agent.run(attachments=[...])`.
3. The LLM calls a filesystem MCP tool with that path to read content.

### Agent → user attachments

The agent signals attachments back to the client by emitting markers in its reply text:

```
[IMAGE:/path/to/chart.png]
[FILE:/path/to/report.pdf]
[VOICE:/path/to/memo.ogg]
[VIDEO:/path/to/clip.mp4]
```

The gateway strips these markers from the response text and delivers them as a structured `attachments: [{type, path, filename}]` array on the WS `response` message. Bridges render them as native media attachments (Telegram photo, Discord file, WhatsApp media).

## Voice Transcription

Voice messages are transcribed automatically. Two backends (tried in order):

1. **faster-whisper** (local, free, no API key) — install with `pip install openagent-framework[voice]`
2. **OpenAI Whisper API** (cloud fallback) — requires `OPENAI_API_KEY` in environment
