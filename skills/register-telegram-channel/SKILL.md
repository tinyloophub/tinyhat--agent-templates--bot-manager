---
name: register-telegram-channel
description: Bind a Telegram bot token to an existing tinyhat agent so users can chat with it. Pairs with provision-user-agent. Pending channels.register; reply 'not yet shipped' until then.
runtime_tier: 0
vault_access:
  - agent
---

# register-telegram-channel

The maintainer has a tinyhat agent (just created or pre-existing)
and wants to wire a Telegram bot to it so end users can chat. This
skill is the "plug the bot into the agent" half of provisioning.

## Prerequisites

- The target agent exists (`agents.list` shows its id).
- The Telegram bot token has been stored in the agent's per-agent
  vault row at `(owner_kind='agent', owner_id=<agent.id>,
  vault_name='telegram')` with key `TELEGRAM_BOT_TOKEN`. Today the
  create-agent flow (`agents.create`) writes this row directly from
  the BotFather token the maintainer pastes into chat; per-agent
  Telegram secrets are NOT user-owned and the `set-credential`
  skill (which targets user vaults via `users.me.vaults.upsert`)
  does NOT touch this row.

## Conversation pattern

1. **Confirm the agent.** "Which agent? `agents.list` shows
   <names>." If only one match in context, name it back as a
   one-line confirmation.
2. **Check the token is set.** There is no shipped endpoint that
   surfaces per-agent vault key names today — `agents.list` carries
   `telegram_bot_username` / `telegram_bot_user_id` on the same row
   when the create-agent flow has bound the bot, so use those as
   the "is this agent already wired?" signal. When
   `channels.register` runs, the platform's response is the source
   of truth on whether the credential was found — pass any error
   through verbatim.
3. **Confirm the channel name.** Default it to the bot's @handle
   (the agent should know its own handle; if not, ask the user).
4. **Echo the plan in one sentence**, wait for yes, then call
   `channels.register` (citation only — see below).
5. **Return the new channel id and the test instructions** —
   typically "send `/start` to @<handle> and see what comes back".

## When the endpoint isn't shipped yet

`channels.register` is on the planned list in
`tinyhat-platform-api-reference`. Until it ships, say:

> Telegram channel registration isn't wired up yet — that endpoint is on the way. Once it lands I'll be able to bind <agent> to <bot-handle>.

Don't fall back to writing rows directly; the manager bot doesn't
have that capability.

## Do not

- Do not call the Telegram API directly. The platform's job is to
  route through `channels.register`, which performs the
  `setWebhook` + bot-validation + vault binding in one transaction.
  A direct call leaves the platform in an inconsistent state.
- Do not publish the bot token in any reply, ever. Confirmation
  text is `"Bot token stored."`, not the token itself.
