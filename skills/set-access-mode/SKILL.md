---
name: set-access-mode
description: Manage who can chat with this agent (the access-mode enum) and who can manage it (per-agent admin tier). Add, update, or remove access-list entries. Confirm-then-act on every mutation.
runtime_tier: 0
---

# set-access-mode

Each agent has an **access mode** that decides who can chat with it
on Telegram, plus a per-agent **access list** that names the
individual users who carry whitelist, blacklist, or admin status.
This skill is the chat-driven surface for every read and mutation
on those two things.

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

## Calling pattern — read

Use these for "who can talk to me right now?" / "show my settings"
intents. Both endpoints are read-only and do not require
confirmation:

- `agents.access-mode.get` — `GET /hapi/v1/agents/<self>/access-mode`
  returns `{agent_id, agent_handle, mode, allowed_modes}`. The
  `allowed_modes` list lets you offer the user a menu of valid
  next states without baking the enum into your prompt.
- `agents.access-list.get` — `GET /hapi/v1/agents/<self>/access-list`
  returns `{agent_id, agent_handle, entries: [{user_id, kind, is_admin,
  added_by_user_id}, …]}`. Use this for "who's on the whitelist?",
  "who can manage me?", or to verify a target user's current state
  before changing it.

Use `<self>` literally — the bot-manager passes its own agent handle
on the path because every flow this skill handles is "manage the
agent the user is talking to right now."

## Calling pattern — set the access mode

Use this for "make yourself public", "switch to invite-only", "go
private", "open to my account team":

1. **Disambiguate the intent.** Plain English maps onto the four
   modes:
   - "go private" / "lock it down" → `restricted`
   - "let my team in" / "open to my account" → `account_members`
   - "invite-only" / "let me pick who" → `whitelist`
   - "make it public" / "open to anyone" / "anyone except blocked" →
     `public_with_blacklist`
   When the user says something ambiguous ("let some friends in"),
   ask one clarifying question — `whitelist` and `account_members`
   are easy to confuse.
2. **Read the current state** with `agents.access-mode.get`. Show
   the user what they're about to change FROM.
3. **Confirm.** Mirror the change back: "I'll switch from `<from>`
   to `<to>` — that means `<plain-language consequence>`. Confirm?"
   Wait for an explicit "yes". The mirror catches the "I meant
   `account_members`" slip, especially around `whitelist` vs
   `public_with_blacklist`.
4. **Apply** with `agents.access-mode.set` — `POST /hapi/v1/agents/<self>/access-mode`,
   body `{"mode": "<one of the four>"}`. The endpoint is idempotent
   on the same value.
5. **Report back.** Echo the new mode. If the user widened to
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

1. **Resolve the target user.** When the user names someone by
   handle ("add alice"), you'll typically need their numeric
   `tinyhat_users.id`. Ask them for the user's id if you don't
   already have it from context, or — if `users.list` is in your
   ACL — search for the handle there. Do not guess. If the
   platform refuses with 404 ("target user not found"), tell the
   user directly: the target needs to exist in the platform's user
   table before they can be added to an access list.
2. **Confirm.** Mirror the change: "I'll add `alice` to the
   whitelist (chat-allowed, NOT admin). Confirm?" or "I'll promote
   `alice` to admin (chat-allowed, can manage settings). Confirm?"
3. **Apply** with `agents.access-list.entries.upsert` —
   `POST /hapi/v1/agents/<self>/access-list/entries`, body
   `{"user_id": <id>, "kind": "allow"|"deny", "is_admin": false|true}`.
   Idempotent on `(agent_id, user_id)`; the same call twice returns
   the same row.
4. **Removing** an entry uses
   `agents.access-list.entries.delete` —
   `DELETE /hapi/v1/agents/<self>/access-list/entries/<user_id>`.
   404 means there was no entry to begin with; surface that as "no
   entry to remove" rather than as an error.
5. **Report back.** Show the new row's shape (kind + admin) so the
   user can see exactly what they have.

### Mutation refused (`409 deny + admin`)

The platform refuses `{"kind": "deny", "is_admin": true}` at both
the service and the database — a denied user cannot be admin. If
the user's intent was contradictory ("block bob but make him
admin"), name the conflict and ask which they meant.

### Permission refused (`403`)

Only an agent **admin** can call any of these mutations. The
caller's identity is determined by the platform from the chat
context — you do NOT need to set any header or say who you are.
If the platform replies `403`, the user is not an admin of this
agent. Tell them plainly:

> You aren't an admin of this agent, so I can't change its access settings. The owner or another admin can promote you with the `promote-to-admin` flow.

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
