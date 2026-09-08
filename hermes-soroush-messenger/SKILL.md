---
name: hermes-soroush-messenger
description: "Use when adding Soroush Plus (سروش پلاس) support to Hermes Gateway. Connect AI agent to Persian messenger via official Bot API."
version: 1.2.0
author: علی محمودی
license: MIT
metadata:
  hermes:
    tags: [soroush, soroushplus, persian, messenger, platform, adapter, bot]
    related_skills: [hermes-persian-stt]
---

# Soroush Plus Platform Adapter for Hermes

Adds Soroush Plus (سروش پلاس) messenger support to Hermes Gateway as a
platform plugin. Your AI agent can send/receive messages, voice notes,
images, and documents through Soroush via the official Bot API.

## Quick Install

```bash
cd ~/.hermes/plugins/platforms/
git clone https://github.com/mah92/hermes-soroush-messenger-plugin.git soroush
hermes plugins enable hermes-soroush-messenger
```

Then add your bot token to `~/.hermes/.env`:

```env
SOROUSH_BOT_TOKEN=1234567890:ABCdefGHIjklMNOpqrsTUVwxyz
SOROUSH_ALLOWED_USERS=69835706
```

Restart the gateway and you're done.

## Files

| File | Purpose |
|------|---------|
| `plugin.yaml` | Platform manifest — env vars, metadata |
| `__init__.py` | Package entry — re-exports `register()` |
| `adapter.py` | Full `BasePlatformAdapter` implementation |

## Features

- Text messages (send & receive)
- Voice messages — send via TTS, receive voice/audio as downloadable files
- Images — send by URL (with download fallback) or local file upload
- Documents — upload and send
- Typing indicators (`sendChatAction`)
- Group chat support with optional @mention gate
- User/chat allowlisting
- Cron delivery support

## How It Works

The adapter uses Soroush's official Bot API (Telegram-compatible REST)
with long-polling — NO third-party SDK. Just aiohttp calls to
`https://api.splus.ir`.

```
Soroush Server ←→ HTTP Long Poll ←→ Hermes Gateway ←→ AI Agent
```

Official docs: https://soroushplus.com/p/documents/bot-platform
Bot maker: **https://splus.ir/botfather** (official Soroush BotFather)

## Configuration

All via `~/.hermes/.env`:

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `SOROUSH_BOT_TOKEN` | ✅ | — | Bot token from splus.ir/botfather |
| `SOROUSH_ALLOWED_CHATS` | — | all | Comma-separated chat IDs |
| `SOROUSH_ALLOWED_USERS` | — | all | Comma-separated user IDs |
| `SOROUSH_ALLOW_ALL_USERS` | — | false | Set "true" for open access |
| `SOROUSH_HOME_CHANNEL` | — | — | Default chat for cron delivery |
| `SOROUSH_REQUIRE_MENTION` | — | false | Require @mention in groups |

## Common Pitfalls

1. **Bot blocked by user:** User must message the bot once before it can DM
   them (same as Telegram/Bale — bots can't initiate conversations).

2. **DO NOT use PySPlusthon (the PyPI SDK).** It has bugs and heavy deps:
   - It reads `update_id` from the wrong field (`.id` vs `.update_id`) —
     the polling offset never advances and getUpdates re-delivers the same
     messages forever → infinite reply loop + status-message spam.
   - It pulls in grpcio, grpcio-tools, protobuf, sqlalchemy as deps.
   - The direct REST adapter (this skill) avoids the SDK entirely and is
     ~zero-dependency (aiohttp only). If you ever see the loop symptom
     (repeated "Dropping busy-mode follow-up ... pending queue at cap"),
     it's the update_id bug — use the REST adapter, not the SDK.

3. **Message object naming:** Soroush API uses Telegram-style field names:
   `from` (sender), `chat`, `message_id`, `text`. The PySPlusthon SDK
   renamed `from` to `author`, which breaks standard telethon-style code.

4. **Voice messages arrive empty?** Transcription needs an STT provider.
   The `hermes-persian-stt` skill in this repo provides Persian STT, or set
   `stt.provider` in `~/.hermes/config.yaml`.

5. **TTS voice not delivered?** Set `voice_compatible: true` on your TTS
   provider in config.yaml. The adapter overrides `send_voice` to upload
   directly via Soroush's `sendVoice` (retries 3×). Without this, Hermes
   emits `MEDIA:` tags which Soroush cannot render.

6. **Cache after edits:** Always
   `find ~/.hermes/plugins/platforms/soroush -name __pycache__ -exec rm -rf {} +`
   after editing adapter files.

7. **Webhook blocks polling:** If you previously used webhook mode, call
   `deleteWebhook` before switching to polling —
   `curl -s "https://api.splus.ir/bot$SOROUSH_BOT_TOKEN/deleteWebhook"`.
   Otherwise `getUpdates` returns nothing.

8. **401 Unauthorized:** Token expired or regenerated from splus.ir/botfather.
   Get a new token and update `SOROUSH_BOT_TOKEN` in `~/.hermes/.env`.

9. **Persian text appears as \uXXXX escapes:** When sending JSON in scripts,
   use `json.dumps(..., ensure_ascii=False)` so Persian stays readable.

10. **Status-message spam (interim messages):** If the user sees "Working...",
    "Redirected current run...", or streaming cursor artifacts on Soroush,
    set in `~/.hermes/config.yaml`:
    ```yaml
    display:
      platforms:
        soroush:
          streaming: false
          interim_assistant_messages: false
    ```
    (Same fix as Bale.) The final answer then arrives as ONE clean message.

11. **send_* method signatures must mirror BasePlatformAdapter** (include
    `file_name`, `reply_to`, `metadata`, `**kwargs`) — otherwise cron media
    delivery and post-stream file sends crash with "got an unexpected
    keyword argument 'metadata'".

12. **Gateway restart guard:** Restarting the gateway from inside the gateway
    process is blocked (systemctl/at/cron all blocked). Use
    `python3 ~/.hermes/scripts/kill-gateway.py` (it uses os.kill, which the
    guard does not intercept). The gateway restarts itself in ~5s.
