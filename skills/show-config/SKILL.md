---
name: show-config
description: Read-only summary of an agent's config from agents.list (model, access mode, owner). Richer fields (harness, vault key names, skills, channels) pending agents.show_config.
runtime_tier: 0
---

# show-config

The owner asks "what's the current setup for <agent>?". This
skill answers with a short, structured summary — and explicitly
flags anything that requires changing through one of the other
skills (so the conversation can chain naturally).

## What to surface

The eventual rich endpoint is `agents.show_config` (planned).
**Today, `agents.list` is the catch-all read** and its row carries:

- **Agent id and name**
- **Owner** (account + primary user from the joined row)
- **Access mode** — one of `restricted` / `account_members` /
  `whitelist` / `public_with_blacklist`. Plain-language gloss when
  reading to the user: "only you" / "your account members" / "you
  plus the people you've added" / "anyone except the people you've
  blocked". The full per-agent list of who's whitelisted /
  blacklisted / admin lives behind `agents.access-list.get` — call
  that one alongside `agents.list` when the user asks about a
  specific person rather than the headline mode.
- **Model** (`model_provider` + `model_name`)
- **Hat id** (so the user knows which hat is bound). The Telegram
  binding lives on `telegram_bot_username` / `telegram_bot_user_id`
  on the same row — render those when they're set.

Render those, and mark every richer field — harness id+version,
vault key names, skills list, bound channels — as "on the way
once `agents.show_config` ships". Do not invent values you don't
have. Vault rows are now keyed on the principal post-#209
(`agents.list` doesn't carry them anymore); to confirm whether a
user has set a credential, call `users.me.vaults.list` (returns
key shape, never values).

## Conversation pattern

1. **Take the agent name.** If ambiguous, list candidates and
   ask which one.
2. **Render the summary** as a short Telegram-formatted message.
   Plain text by default; one block, no headings.
3. **Suggest one or two natural next steps.** "Want to change
   the model? Just say so." This turns the read into a starting
   point for the next workflow.

## When the rich endpoint isn't shipped yet

`agents.show_config` is planned. Until it ships, the today path
above (`agents.list` filtered to the requested agent) is the
most you can render. Be explicit about the gap in your reply,
e.g. "I can see the model and access mode; harness, vault key
names, skills, and bound channels are on the way once
`agents.show_config` ships."

## Do not

- Do not surface vault values, ever. The endpoint summary is
  designed to give names only; if a future version returns more,
  keep showing only the names.
- Do not invent fields. If a value isn't in the response, omit
  the line; don't write "harness: ?" or "model: unknown".
- Do not turn show-config into an audit. The user wanted a quick
  status snapshot, not a 30-line trace. Keep it tight.
