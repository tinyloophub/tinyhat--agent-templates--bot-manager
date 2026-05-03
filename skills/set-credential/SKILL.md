---
name: set-credential
description: Store a credential in the chatting user's vault so every agent they own can use it. Stash via users.me.vaults.upsert; never echo the secret back.
runtime_tier: 0
vault_access:
  - user
---

# set-credential

Users store **credentials** — OpenAI keys, GitHub PATs, Slack
tokens — once, on themselves, and every agent they own (bot-manager
+ any user-created agent) can use them. The chat-first surface for
that is `POST /hapi/v1/users/me/vaults`; this skill is the
conversation pattern that walks the user through it.

This is the **only** skill the bot-manager has that takes a secret
as input.

## Where the credential lives

The platform stores user-owned credentials at
`(owner_kind='user', owner_id=<chatting user>, vault_name='<bag>')`.
Each vault row is one named bag of `KEY=VALUE` entries — for example,
`vault_name='openai'` with `OPENAI_API_KEY=sk-…`. A user can have
many vault rows, one per upstream service.

`gated_api_call` reads from the user vault when an agent's outbound
ACL entry says `owner_kind: user`. Per-agent secrets (Telegram bot
tokens, webhook secrets) keep living in the agent's own
`(agent, agent.id, vault_name)` row — those are not user-owned and
this skill does not touch them.

## Conversation pattern

1. **Confirm what the user wants to stash.** Two pieces:
   - **The vault name** (e.g. `openai`, `github`, `slack`) — the
     bag the entry lives in.
   - **The key name** (e.g. `OPENAI_API_KEY`) — uppercase-snake-case
     by convention, matches the env-var the upstream's docs name.
2. **Take the secret.** This is the one place the user types a
   credential into chat. **Do NOT echo it back, ever.** As soon as
   you receive it, replace any local mention of it with `***` in
   anything you say.
3. **Apply** by calling `gated_api_call` against
   **`users.me.vaults.upsert`** (cited per `HAT.md`'s "always
   cite by operationId" rule — see
   `tinyhat-platform-api-reference` for the surface). Body:

   ```json
   {
     "vault_name": "<vault>",
     "env": {"<KEY>": "<value>"}
   }
   ```

   The platform server-injects `X-Tinyhat-User-Id` from the
   conversation's authenticated user — you do **not** know or set
   this header, and any value you put in the request is stripped
   before the call leaves the platform. The endpoint MERGES the
   entry into the existing row (other keys in the bag survive) and
   returns the post-write key shape — keys only, never values.
4. **Report back** with **only** the redacted confirmation:
   "Stored. Your `<vault>` vault now has `<KEY>` set." Don't
   restate the value, even partially, even on a mistype.

## Clearing a credential

If the user wants to remove an entry, send the same
`users.me.vaults.upsert` body with an empty string for the value
(`{"<KEY>": ""}`) — empty values are treated as deletes by the
endpoint. The response confirms the removal.

For surgical key removal use `users.me.vaults.delete_key`; to drop
the whole vault row use `users.me.vaults.delete`.

## Do not

- Do not echo the secret back, in any form, in any reply, ever.
  Not "you typed `sk-...abc`", not "I'll store `xxxxx`". Ever.
- Do not write user-owned credentials into the **agent's** vault.
  The agent vault is for per-agent secrets the platform manages
  (Telegram bot tokens, webhook secrets); user keys belong on the
  user. Routing `OPENAI_API_KEY` into the agent vault would lock
  the user to one agent's view of their own credential.
- Do not store a secret with a key name reserved for the platform
  (`TINYLOOP_*`, `TELEGRAM_*` are platform-reserved). The endpoint
  doesn't enforce this today, but the convention keeps user keys
  from colliding with platform-managed slots.
