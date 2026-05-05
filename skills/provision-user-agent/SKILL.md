---
name: provision-user-agent
description: Create a user-owned Tinyhat agent through Telegram Managed Bots, then call agents.create with managed_bot_id and no user-visible token.
runtime_tier: 0
---

# provision-user-agent

The user wants a new Tinyhat agent they can chat with in Telegram.
This is the zero-token Managed Bots flow: propose a Telegram
new-bot link, wait for Telegram's matched `managed_bot`
confirmation, then create the Tinyhat agent with `managed_bot_id`.

Never ask the user for a BotFather token. Never echo, paste, or
handle a token in chat. The backend fetches the managed bot token
server-side through Telegram `getManagedBotToken` and writes it only
to the new agent's own `telegram` vault.

## Prerequisites

- The platform's bot-manager bootstrap has run (`infrastructure.get`
  returns `bootstrapped: true` for the bot-manager).
- The manager bot has Telegram Bot Management Mode enabled:
  `getMe.can_manage_bots=true`. If the proposal tool reports this is
  missing, say the manager bot needs Bot Management Mode enabled once
  before it can create managed bots.
- The chatting Telegram user is the creator. They must have sent
  `/start` to the manager bot at least once so Tinyhat can bind their
  Telegram identity to a Tinyhat user row.
- `agents.create` is called through `gated_api_call` with the
  operationId documented in `tinyhat-platform-api-reference`.

## Names are separate

- Tinyhat account handle: the account/organization slug, for example
  `tinyhat` or `acme`.
- Tinyhat agent handle: account-scoped, kebab-case, for example
  `support` in `acme/agents/support`. It can be short and friendly.
- Telegram bot username: globally unique on Telegram, 5-32
  letters/digits/underscores, starts with a letter, and ends with
  `bot` case-insensitively, for example `acme_support_2026_bot`.
- Telegram display name: the human-readable bot name shown in
  Telegram, for example `Acme Support`.

Do not reuse a Tinyhat kebab-case handle directly as a Telegram
username. Ask for both when needed. Telegram's confirmation sheet is
the source of truth for whether the global Telegram username is
available; smarter candidate generation/retry is a separate
follow-up.

## Conversation pattern

1. **Read the request.** Extract the purpose, account/organization,
   desired Tinyhat handle, Telegram display name, and Telegram
   username if present.
2. **Ask for missing facts once.** Keep it concise. Example:
   "I can do that. What Tinyhat account should own it, what easy
   in-account handle do you want, and what Telegram username should
   I propose? The Telegram username must end in `bot`; it can differ
   from the Tinyhat handle."
3. **Reject import-existing-bot requests.** If the user wants to
   paste a token, import an existing third-party bot, or wire a bot
   they already created outside this flow, say that import is not the
   Managed Bots create flow yet. Do not ask for a token.
4. **Confirm the plan.** One sentence, then wait for an explicit yes:
   "I'll propose Telegram bot `@acme_support_2026_bot` named
   `Acme Support`, create Tinyhat agent `acme/agents/support`, and
   make you the initial restricted owner/admin. You can add people or
   change access mode later."
5. **Propose the Telegram managed bot.** Call the function tool:
   `propose_managed_bot_creation(suggested_username, suggested_name)`.
   Use the Telegram username and display name from the confirmed plan.
6. **Send the deep link.** Reply with the returned `deep_link` and
   say: "Tap this, keep the suggested username unchanged, confirm in
   Telegram, then reply here when you see the confirmation and I'll
   finish the setup." Telegram does not echo Tinyhat's correlation id
   back, so changing the username can make the callback unmatched.
7. **Finish after the callback is confirmed.** Once the conversation
   has a matched confirmation with the numeric `managed_bot_id`, call
   `agents.create` through `gated_api_call`:

   ```json
   {
     "channel": "telegram",
     "kind": "user",
     "account_handle": "acme",
     "handle": "support",
     "name": "Acme Support",
     "managed_bot_id": "777001234"
   }
   ```

   Do not include `telegram_bot_token`. Do not set
   `owner_telegram_handle` unless you are only asserting the same
   Telegram handle from the confirmed managed-bot creator; the
   platform rejects mismatches.
8. **Report the result.** Tell the user the Tinyhat handle and the
   Telegram address from the create response or provisioning
   metadata. Suggest sending the new bot a first message.

## Error handling

- **Invalid Telegram username:** ask for a Telegram-safe username
  ending in `bot`. Explain that the Tinyhat handle can stay simpler.
- **Bot Management Mode missing:** say the manager bot needs Telegram
  Bot Management Mode enabled once before Tinyhat can create managed
  bots. Stop; do not fall back to BotFather-token UX.
- **Unmatched managed-bot event:** explain that Telegram confirmed a
  bot, but Tinyhat could not match it to this proposal. Ask the user
  to restart the Managed Bots flow from this chat, or hand off to
  operator review with the event id.
- **No confirmed row for `managed_bot_id`:** tell the user to finish
  the Telegram confirmation first and retry with the same managed bot.
- **Owner mismatch:** explain that the initial owner must be the
  Telegram user who created the managed bot. The creator can later add
  people or promote admins through access settings.

## Do not

- Do not ask for a BotFather token in the user-agent flow.
- Do not call `agents.create(kind='user')` with
  `telegram_bot_token`.
- Do not let the requester assign initial ownership to someone other
  than the Telegram user who created the managed bot.
- Do not call `register-telegram-channel` after this flow. Managed
  bot agents are bound automatically during `agents.create`.
- Do not skip the confirmation step before proposing the bot or
  before creating the Tinyhat agent.
