---
name: set-access-mode
description: Switch a tinyhat agent between public, invite-only, and private modes; drives the whitelist gate the webhook enforces. Pending the endpoint; reply 'not yet shipped' until then.
runtime_tier: 0
---

# set-access-mode

Each agent has an **access mode** that decides who can talk to it
on Telegram. The three modes are:

- `public` — anyone who finds the bot can use it.
- `invite_only` — only Telegram users on the whitelist; everyone
  else gets the canned invite-request reply (issue #98 / #105).
- `private` — only the owner and account members can use it;
  everyone else is silently ignored.

The default for newly-provisioned agents is `invite_only` so the
maintainer doesn't accidentally publish a public bot.

## Conversation pattern

1. **Confirm the agent.** As elsewhere, name it back if there's
   any ambiguity.
2. **Confirm the new mode.** If the user said "make it public",
   echo back: "I'll switch <agent> to public — anyone who finds
   it on Telegram will be able to use it. Confirm?" The mirror
   helps catch the "I meant private" slip.
3. **Apply.** Call `agents.set_access_mode` (cited in the planned
   list of `tinyhat-platform-api-reference`). Body shape lands with
   the endpoint; do not invent it before then.
4. **Report back** with the new mode and whose-allowed summary
   (e.g. "switched to invite_only; 3 users on the whitelist").

## When the endpoint isn't shipped yet

`agents.set_access_mode` is on the planned list. Until it ships,
you can read the current mode from `agents.list` (each row carries
an `access_mode` field) but you can't change it. Say:

> The current mode is <mode>. Changing it isn't wired up yet — the endpoint that does the swap is on the way.

Do NOT modify the database directly (you can't anyway; the
manager bot has no DB access except via the platform API).

## Do not

- Do not advertise public mode to a user who hasn't asked for it.
  The default is invite-only on purpose — opening a bot to the
  public is a one-way trust decision.
- Do not assume the user understood the difference between
  `invite_only` and `private`. "Only my friends can use it" maps
  to either — ask which one they want.
