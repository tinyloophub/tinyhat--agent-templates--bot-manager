---
name: set-access-mode
description: Manage who can chat with a target agent (the access-mode enum) and who can manage it (per-agent admin tier). Add, update, or remove access-list entries. Confirm-then-act on every mutation.
runtime_tier: 0
---

# set-access-mode

Each agent has an **access mode** that decides who can chat with it
on Telegram, plus a per-agent **access list** that names the
individual users who carry whitelist, blacklist, or admin status.
This skill is the chat-driven surface for every read and mutation
on those two things for the **target agent** the user is asking
about. The bot-manager is only one agent; do not assume every
access-mode request targets bot-manager.

The platform endpoints this skill calls are documented in
[`tinyhat-platform-api-reference`](../tinyhat-platform-api-reference/SKILL.md).
Every call here is a `gated_api_call` — there are no in-sandbox curls
and you never see a token.

## Access modes — the four-value closed enum

The platform's access gate decides who can chat using these four
values, in order from most restrictive to most open:

| Mode | Who can chat |
| --- | --- |
| `restricted` | Only the agent's primary owner. Everyone else is silently ignored. |
| `account_members` | Primary owner plus every member of the agent's owner account. |
| `whitelist` | Primary owner plus every user with a `kind='allow'` entry on the access list. |
| `public_with_blacklist` | Anyone, except users with a `kind='deny'` entry on the access list. |

Newly-provisioned agents land in `restricted`; widening is an
explicit owner action. The platform refuses any other value.

When you talk to a user, prefer plain language: "only you", "your
account members", "you plus the people you've added", "anyone except
the people you've blocked". Show the canonical mode value in
parentheses or in the confirmation prompt — users frequently want
to know the exact name they'll see in the admin panel.

## Per-agent admin tier

Independent of who can **chat**, each agent has a per-agent **admin
tier** that decides who can manage the agent's settings (flip the
mode, add or remove access-list entries, promote others to admin).
The agent's primary owner is **always** admin without an explicit
entry. Anyone else who needs admin rights gets a
`kind='allow', is_admin=true` row on the access list. A denied user
cannot also be admin — the platform refuses the
`kind='deny' AND is_admin=true` combination.

The platform also short-circuits to admin for users with the
tinyloop-wide superadmin flag (the maintainer's identity in dev /
prod). You don't need to know who that is — `agents.access-mode.set`
will simply succeed for them and fail with `403` for anyone else who
isn't on the admin tier.

## Identifying the target agent on the URL

Every endpoint below is path-parameterised on the **target agent
identifier**.
The platform's resolver accepts three shapes:

- a numeric `tinyhat_agents.id`, or
- the canonical handle `<account-slug>/agents/<agent-name>`, or
- the slashless handle `<account-slug>--agents--<agent-name>`.

A literal placeholder string (e.g. `<self>`, `<this>`, `me`,
`{agent_identifier}`, `{resolved_agent_identifier}`) will 404.
Resolve the target agent first from the user's request:

1. If the user gives a canonical handle like
   `tinyloop/agents/acme-helper`, convert it to the slashless URL
   identifier `tinyloop--agents--acme-helper` before calling HAPI.
2. Otherwise call `agents.list.hapi` and match by the agent name
   or Telegram binding the user named. If there is more than one
   match, ask which agent they meant.
3. If the user says "this bot", "you", or "bot-manager", clarify
   whether they mean the manager bot itself or one of their agents.
   Only use `tinyhat/agents/bot-manager` when the user explicitly
   chose the platform bot-manager agent.

After resolution, replace the route variable with the resolved
identifier. Prefer the slashless handle when you have one. Example:
for the target handle
`tinyloop/agents/acme-helper`, the access-mode read URL is
`/hapi/v1/agents/tinyloop--agents--acme-helper/access-mode`.
The `/agents/{ex_id_or_handle:path}/...` route is generic for all
agents; slash-bearing canonical handles still work, but the
slashless form keeps the URL shape easier to read.

## Agent-admin and target-user scope

Every endpoint in this skill is limited to admins of the **target
agent**. The platform injects the chatting user as the acting user
before the request reaches `/hapi/v1`; you do not set
`X-Tinyhat-Acting-User` yourself. A 403 means this chatting user is
not an admin of that target agent.

For access-list mutations, a `user_id` is not a global lookup
escape hatch. Use a target `tinyhat_users.id` only when it came from
agent-scoped context, such as:

- an existing access-list row for the target agent,
- a pending / recent inbound request for the target agent, or
- an agent-scoped platform response that explicitly ties the user
  to the target agent.

Do not call a global user list or ask the owner to guess arbitrary
database ids. If you only have a Telegram handle and no agent-scoped
user id yet, ask the person to message the target agent first or say
you need an agent-scoped lookup before you can add them.

## Calling pattern — read

Use these for "who can talk to me right now?" / "show my settings"
intents. Both endpoints are read-only and do not require
confirmation:

- `agents.access-mode.get` — `GET /hapi/v1/agents/{resolved_agent_identifier}/access-mode`
  returns `{agent_id, agent_handle, mode, allowed_modes}`. The
  `allowed_modes` list lets you offer the user a menu of valid
  next states without baking the enum into your prompt.
- `agents.access-list.get` — `GET /hapi/v1/agents/{resolved_agent_identifier}/access-list`
  returns `{agent_id, agent_handle, entries: [{user_id, kind, is_admin,
  added_by_user_id}, …]}`. Use this for "who's on the whitelist?",
  "who can manage `<agent>`?", or to verify a target user's current state
  before changing it. Do not send `{resolved_agent_identifier}`
  literally; replace it with the numeric id or slashless handle you
  resolved above.

## Calling pattern — set the access mode

Use this for "make yourself public", "switch to invite-only", "go
private", "open to my account team":

1. **Resolve and confirm the target agent.** If the user said
   "make acme-helper public", resolve `acme-helper` first and name
   it back. If they said "make yourself public", ask whether they
   mean the bot-manager or another agent.
2. **Disambiguate the intent.** Plain English maps onto the four
   modes:
   - "go private" / "lock it down" → `restricted`
   - "let my team in" / "open to my account" → `account_members`
   - "invite-only" / "let me pick who" → `whitelist`
   - "make it public" / "open to anyone" / "anyone except blocked" →
     `public_with_blacklist`
   When the user says something ambiguous ("let some friends in"),
   ask one clarifying question — `whitelist` and `account_members`
   are easy to confuse.
3. **Read the current state** with `agents.access-mode.get`. Show
   the user what they're about to change FROM.
4. **Confirm.** Mirror the change back: "I'll switch `<agent>` from `<from>`
   to `<to>` — that means `<plain-language consequence>`. Confirm?"
   Wait for an explicit "yes". The mirror catches the "I meant
   `account_members`" slip, especially around `whitelist` vs
   `public_with_blacklist`.
5. **Apply** with `agents.access-mode.set` —
   `POST /hapi/v1/agents/{resolved_agent_identifier}/access-mode`,
   body `{"mode": "<one of the four>"}`. The endpoint is idempotent
   on the same value. Replace `{resolved_agent_identifier}` first.
6. **Report back.** Echo the target agent and new mode. If the user widened to
   `whitelist` or `public_with_blacklist` and the access list is
   empty, mention it — "you're now on `whitelist` but no one is on
   the list yet, so only you can chat. Want me to add anyone?"

## Calling pattern — manage the access list

Use this for "add alice to the whitelist", "block bob",
"promote charlie to admin", "remove alice from the list".

The list is a flat set of `(user_id, kind, is_admin)` rows, where
`kind` is `'allow'` or `'deny'` and `is_admin` is independent. The
upsert endpoint takes both fields together, so a single call can
add a user as a chat-allowed admin, swap them from allow to deny,
or strip admin while keeping chat access.

Steps for a mutation:

1. **Resolve the target agent.** Name it back before changing its
   list.
2. **Resolve the target user from agent-scoped context.** When the
   user names someone by handle ("add alice"), you need a numeric
   `tinyhat_users.id` that is tied to this target agent. Use an
   existing access-list row, pending / recent inbound request, or
   other agent-scoped platform response. Do not use global user
   enumeration and do not accept a guessed id. If the platform
   refuses with 404 ("target user not found") or says the target
   user is outside the target agent's scope, tell the user the
   person needs to message this agent first (or wait for an
   agent-scoped lookup surface) before they can be added.
3. **Confirm.** Mirror the change: "I'll add `alice` to
   `<agent>`'s whitelist (chat-allowed, NOT admin). Confirm?" or
   "I'll promote `alice` to admin on `<agent>` (chat-allowed, can
   manage settings). Confirm?"
4. **Apply** with `agents.access-list.entries.upsert` —
   `POST /hapi/v1/agents/{resolved_agent_identifier}/access-list/entries`,
   body
   `{"user_id": <id>, "kind": "allow"|"deny", "is_admin": false|true}`.
   Idempotent on `(agent_id, user_id)`; the same call twice returns
   the same row.
5. **Removing** an entry uses
   `agents.access-list.entries.delete` —
   `DELETE /hapi/v1/agents/{resolved_agent_identifier}/access-list/entries/<user_id>`.
   404 means there was no entry to begin with; surface that as "no
   entry to remove" rather than as an error.
6. **Report back.** Show the target agent and the new row's shape
   (kind + admin) so the user can see exactly what they have.

### Mutation refused (`409 deny + admin`)

The platform refuses `{"kind": "deny", "is_admin": true}` at both
the service and the database — a denied user cannot be admin. If
the user's intent was contradictory ("block bob but make him
admin"), name the conflict and ask which they meant.

### Permission refused (`403`)

Only an agent **admin** can call any of these reads or mutations.
The caller's identity is determined by the platform from the chat
context — you do NOT need to set any header or say who you are.
If the platform replies `403`, the user is not an admin of the
target agent. Tell them plainly:

> You aren't an admin of that agent, so I can't read or change its access settings. The owner or another admin can promote you with the `promote-to-admin` flow.

Do not retry. Do not pretend the call worked.

## Confirm-then-act is mandatory

Every mutation in this skill — mode flip, access-list add /
update / delete, admin promote / demote — must be preceded by a
confirmation turn from the user. No silent state changes on a
single user turn. The mode flip is especially load-bearing:
flipping into `public_with_blacklist` opens the bot to every
Telegram user who can find it, and unwinding that decision means
adding everyone who's now talking to the bot to the deny list.

## Do not

- **Do not invent mode names** like `public`, `invite_only`, or
  `private` — those were the legacy names. The platform only
  recognises the four canonical values listed above. If the user
  uses the legacy names, translate them but echo back the canonical
  value in the confirmation.
- **Do not advertise `public_with_blacklist` to a user who hasn't
  asked for it.** It's the most-open mode and reverting it is
  costly.
- **Do not set the `X-Tinyhat-Acting-User` header yourself.** The
  platform's `gated_api_call` gate server-injects the acting user
  from the chat context. Setting the header from inside the
  sandbox would either be ignored by the gate or rejected; either
  way you should not try.
- **Do not bypass the platform.** This bot has no database access;
  every state change goes through `gated_api_call`.
- **Do not change the mode as a side effect of another change.**
  Even when "lock it down" is implied (e.g. the user is asking to
  remove someone), confirm the mode flip explicitly.
