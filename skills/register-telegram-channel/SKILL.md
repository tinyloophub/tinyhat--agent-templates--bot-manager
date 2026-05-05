---
name: register-telegram-channel
description: Future import-existing-bot flow. Managed Bots agents created by provision-user-agent are already bound during agents.create.
runtime_tier: 0
vault_access:
  - agent
---

# register-telegram-channel

This skill is only for a future import-existing-bot escape hatch:
binding a pre-existing third-party Telegram bot to an existing
Tinyhat agent.

It does **not** apply to agents created through
`provision-user-agent`. Managed Bots agents are bound automatically
during `agents.create(kind='user')`: the backend fetches the
managed-bot token server-side, writes it to the new agent's own
`telegram` vault, and registers the per-agent webhook in the same
provisioning flow.

## Prerequisites

- The target agent exists.
- The platform ships an import/bind endpoint for existing bots.
  Today `channels.register` is still planned, not live.
- A token, when this future flow exists, must be stored through a
  platform vault workflow that never echoes the token back in chat.

## Conversation pattern

1. **First classify the request.** If the user is creating a new
   Tinyhat user agent, switch to `provision-user-agent`; do not use
   this skill.
2. **If the user already has a third-party bot, defer.** Say:
   "Importing an existing Telegram bot is not wired up yet. New
   Tinyhat user agents use Telegram Managed Bots instead, so you
   don't have to paste a token."
3. **When the endpoint ships,** confirm the target agent, confirm
   the import intent, then call `channels.register` through
   `gated_api_call`. Until then, do not invent a substitute.

## When the endpoint isn't shipped yet

`channels.register` is on the planned list in
`tinyhat-platform-api-reference`. Until it ships, say:

> Importing an existing Telegram bot isn't wired up yet. New Tinyhat user agents use Telegram Managed Bots, so I can create a fresh managed bot for you without asking for a token.

Don't fall back to writing rows directly; the manager bot doesn't
have that capability.

## Do not

- Do not use this skill after `provision-user-agent`. Managed-bot
  agents are already registered.
- Do not ask for, paste, or echo a Telegram bot token.
- Do not call Telegram directly. The platform must own webhook
  registration and vault writes.
