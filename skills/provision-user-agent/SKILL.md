---
name: provision-user-agent
description: Walk a maintainer through creating a new user-owned tinyhat agent; confirms hat, owner, token, then calls agents.create. Pending the endpoint; reply 'not yet shipped' until then.
runtime_tier: 0
---

# provision-user-agent

The maintainer wants to spin up a new agent for a user (their own
account, or someone they're inviting). This skill walks the
conversation: gather the few facts the platform needs, confirm
once, then call the platform API.

## Prerequisites

- The platform's bot-manager bootstrap has run (`infrastructure.get`
  returns `bootstrapped: true` for the bot-manager).
- The maintainer is a Tinyloop admin (the API gate enforces this;
  if a non-admin asks, refuse politely without revealing why).

## Conversation pattern

1. **Read the request.** The maintainer might say "create an agent
   for Anna" or "spin up a new bot called acme-helper". Treat the
   bot's intended *handle* as the most important field.
2. **Ask for the missing facts** (one short message, all in one
   go — don't drip-feed):
   - Hat name (the slug; lowercase, kebab-case).
   - Owner — the Telegram username or email of the person it
     belongs to.
   - Telegram bot token? If the user already has one, take it via
     `set-credential` *before* calling `agents.create`. If they
     don't, point them at @BotFather and pause.
   - Harness — default is the platform's current harness; only
     ask if the user explicitly wants a different one.
3. **Echo the plan back in one sentence** ("I'll create
   `acme-helper` owned by @anna, on the default harness, paired
   with the bot token you just set.") and wait for a yes.
4. **Call the API.** Cite operationId `agents.create` from
   `tinyhat-platform-api-reference`. Body shape: TBD until the
   endpoint lands.
5. **Report back** with the new agent's id and the next-step the
   user needs to take (typically: register a Telegram channel via
   `register-telegram-channel`).

## When the endpoint isn't shipped yet

`agents.create` is on the planned-operations list in
`tinyhat-platform-api-reference`. Until it ships, do NOT call any
other endpoint as a substitute. Say:

> Provisioning a new agent isn't wired up yet — the platform endpoint is on the way. I'll be able to do this once issue #102's follow-up lands.

Then stop. Do not invent endpoints, do not pretend the agent was
created, do not write to the database directly.

## Do not

- Do not create the Telegram bot for the user. They go to
  @BotFather themselves; we just take the token.
- Do not ever paste a token into your reply. The vault is the only
  place tokens live; the user sees confirmation that it was
  stored, never the token itself.
- Do not skip the confirmation step in §3. Provisioning is a
  multi-second operation that affects billing once we're past v1;
  a one-message confirmation is cheap and the rollback isn't.
