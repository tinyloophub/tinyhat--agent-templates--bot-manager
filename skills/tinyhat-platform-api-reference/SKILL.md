---
name: tinyhat-platform-api-reference
description: Manpage of Tinyhat platform endpoints callable via gated_api_call. The only callable surface is /hapi/v1/. Cite endpoints by operationId. Read BEFORE calling.
runtime_tier: 0
vault_access:
  - user
  - agent
---

# tinyhat-platform-api-reference

This skill teaches you how to call the Tinyhat platform's internal
API. Auth is automatic — your `gated_api_call` function tool injects
the right Bearer header for you from the bot-manager's vault. You
do **not** need an API key in your message, and you must **never**
ask the user for one.

You also do not need a platform base URL or sandbox bearer token.
For first-party Tinyhat calls, pass a relative `/hapi/v1/...` path;
the backend function tool resolves the platform host server-side.

The only callable surface is **`/hapi/v1/`** (the agent-centric
platform API shipped in #155 / W-REVISE). Do not call any other
path prefix. If a capability you need isn't here yet, say so to
the user — do not invent or guess a URL.

Every other bot-manager skill that needs to mutate the platform
cites endpoints in this file by **`operationId`** (e.g.
`agents.create`, `agents.list.hapi`). Always cite the operationId,
not the URL — paths can rename without bumping the id, and breaking
that indirection is what makes a six-month-old skill stop working.

## The calling pattern

Every call goes through one tool, with one shape:

```text
gated_api_call(
  method   = "GET" | "POST" | "PATCH" | "PUT" | "DELETE",
  url      = "/hapi/v1/<path>",
  json_body= { … } | null,    # only for POST / PATCH / PUT
  query    = { … } | null,    # for GET filters or paging
)
```

Returns either an upstream response shape:

```text
{ "status": <int>, "headers": {…}, "body": <json|text>, "latency_ms": <int> }
```

…or one of the gate's structured errors (decide what to do based
on which one you got):

| Error code             | Meaning + how to react                                  |
|------------------------|---------------------------------------------------------|
| `not_authorized`       | Host or path is not in this agent's ACL. Tell the user the action isn't permitted; do not retry. |
| `credential_missing`   | Operator hasn't set the vault key yet. Tell the user the platform isn't bootstrapped for this action. |
| `vault_unavailable`    | Infrastructure problem (key rotation, MAC mismatch). Apologise briefly and suggest retrying later. |
| `rate_limited`         | Wait `retry_after_seconds` and try again, OR tell the user to retry shortly. |
| `timeout`              | Upstream slow. Try once more if the action is idempotent (GET, list); otherwise ask the user to retry. |
| `upstream_unreachable` | Network flake or platform down. Tell the user; do not loop. |

Never include a credential in the `headers` argument. The gate
strips caller-supplied auth headers before the request leaves our
backend, so doing so is silently ineffective — but more importantly
it signals to the audit reviewer that the agent thinks it owns the
auth, which it does not.

## /hapi/v1 — the only callable surface

Each operation below lives under `/hapi/v1/<path>`. Auth: Bearer
header injected by the gate from the bot-manager's admin vault.

### Bot-manager function tools

These are not HTTP endpoints. They are backend-hosted function tools
from the `bot-manager-20260427` toolkit. Use them directly when the
workflow calls for them.

#### `propose_managed_bot_creation(suggested_username, suggested_name)`

Prepare a Telegram Managed Bots creation link for the chatting user.
Inputs:

- `suggested_username`: Telegram bot username to propose. It must be
  5-32 letters/digits/underscores, start with a letter, and end with
  `bot` case-insensitively.
- `suggested_name`: Telegram display name.

The tool validates the username shape, verifies the manager bot has
Bot Management Mode (`can_manage_bots=true`), stores a pending
correlation row, and returns:

```json
{
  "correlation_id": "mbc_...",
  "manager_bot_username": "tinyhatbot",
  "suggested_username": "acme_support_2026_bot",
  "suggested_name": "Acme Support",
  "deep_link": "https://t.me/newbot/tinyhatbot/acme_support_2026_bot?name=Acme+Support"
}
```

Send the returned `deep_link` to the user and tell them to keep the
suggested username unchanged in Telegram's confirmation sheet.
Telegram does not echo Tinyhat's correlation id back in the later
`managed_bot` update, so the platform matches the callback by
manager, creator, and suggested username.

Matched callbacks are recorded as `status='confirmed'` with creator
identity, managed bot id, managed bot username, and the raw Telegram
confirmation payload. Unmatched callbacks are stored as
`status='unmatched'` for audit/retry only; `agents.create(kind='user')`
will not accept them.

### Agent lifecycle

#### `agents.create` — `POST /hapi/v1/agents`

Create an agent end-to-end (the **same** endpoint the maintainer
uses from `/admin/that` to create the platform agent + every
subsequent user agent).

The credential shape depends on `kind`.

System agent bootstrap (the platform's own bot-manager) uses the
one-time BotFather token:

```json
{
  "channel": "telegram",
  "account_handle": "tinyhat",
  "kind": "system",
  "telegram_bot_token": "<BotFather token>",
  "owner_telegram_handle": "<optional Telegram @username>"
}
```

User-agent creation uses Telegram Managed Bots and **never** accepts
`telegram_bot_token`:

```json
{
  "channel": "telegram",
  "account_handle": "acme",
  "kind": "user",
  "handle": "support",
  "name": "Acme Support",
  "managed_bot_id": "777001234"
}
```

Rules for `kind='user'`:

- `managed_bot_id` is required and must be Telegram's numeric
  `ManagedBotUpdated.bot.id`.
- A matched `tinyhat_managed_bot_creations.status='confirmed'` row
  must exist for that managed bot id. `status='unmatched'` is not
  enough; treat it as retry/operator-review state.
- `telegram_bot_token` is forbidden. The backend calls
  `getManagedBotToken` with the manager token, then writes the
  fetched child token only to the new agent's own `telegram` vault.
- Initial ownership comes from the Telegram user in the confirmed
  `managed_bot` update. If `owner_telegram_handle` is supplied, it
  is only an assertion and is rejected if it does not match the
  confirmed creator.
- `handle` is the Tinyhat in-account agent handle. It is separate
  from the globally-unique Telegram bot username.

The platform persists non-secret bot metadata with the provisioning
state and agent row: Telegram bot id, username, webhook URL, and the
derived `https://t.me/<username>` address. The raw Telegram
confirmation payload is preserved on the managed-bot creation row.

Response:
`{"ex_id": "agt_…", "handle": …, "account_handle": …, "kind": …, "telegram_username": …, "webhook_url": …, "access_mode": …, "status": …}`.
`Idempotency-Key` header dedupes retries against the provisioning
state machine (#176 PR-B-1).

New user agents start `access_mode='restricted'`,
`primary_owner_user_id` set to the managed-bot creator, and an
explicit access-list row for that creator:
`{"kind": "allow", "is_admin": true}`. Only the creator can chat
initially, and because they are admin they can later use
`agents.access-mode.*` and `agents.access-list.*` to widen access or
add more admins.

#### `agents.list.hapi` — `GET /hapi/v1/agents`

Page the platform's `tinyhat_agents` rows. Each row is enriched
with `ex_id`, `webhook_url`, `telegram_bot_username`, and the
owner's Telegram handle (so you don't need a second call to format
an agent description). Query params: `limit` (default 100, max
500), `offset`.

#### `agents.delete` — `DELETE /hapi/v1/agents/{ex_id_or_handle}`

Hard-delete an agent. For `kind=user` agents the per-agent hat
row + telegram vault row are also dropped (releasing the
globally-unique hat name); for `kind=system` only the agent row
is removed (the bot-manager's shared hat / vault stay put — they
belong to the platform bootstrap). The GitHub fork is NOT
touched.

### Per-agent Telegram channel

#### `agents.hooks.telegram` — `POST /hapi/v1/agents/{ex_id_or_handle}/hooks/telegram`

The inbound webhook receiver Telegram itself calls. **You will
never call this** — it's not in your ACL and the secret check is
the per-agent vault row, not the bot-manager's admin Bearer.
Documented here so you can answer "where do my updates land?".

#### `agents.hooks.telegram.reset` — `POST /hapi/v1/agents/{ex_id_or_handle}/hooks/telegram/reset`

Re-register the agent's webhook with Telegram. Reads the agent's
bot token from its per-agent vault, mints a fresh
`TELEGRAM_WEBHOOK_SECRET` (server-side only — never returned),
calls Telegram `setWebhook`. Atomic: vault is only rotated AFTER
Telegram accepts; if the vault write fails, Telegram is rolled
back to the prior secret so inbound delivery never 401s.

#### `agents.telegram.info` — `GET /hapi/v1/agents/{ex_id_or_handle}/telegram/info`

Return Telegram `getMe` + `getWebhookInfo` for the agent. Use this
when the user asks "is my bot working?" or "where is Telegram
sending updates right now?".

### Upstream sync

#### `agents.upstream.status` — `GET /hapi/v1/agents/{ex_id_or_handle}/upstream/status`

Whether the agent's repo is up-to-date with its upstream system
repo. Returns the pinned commit, the upstream HEAD, and a
fast-forward summary.

#### `agents.upstream.sync` — `POST /hapi/v1/agents/{ex_id_or_handle}/upstream/sync`

Fast-forward the agent's repo to upstream HEAD. No body.

### Per-agent access mode + access-list

These five operationIds replace the previously-planned
`agents.set_access_mode`. They are generic for **any** agent whose
identifier resolves through `/hapi/v1/agents/{ex_id_or_handle:path}`;
bot-manager is only the platform's own agent, not a special route
shape. The identifier may be a numeric `tinyhat_agents.id`, the
canonical handle (`tinyhat/agents/bot-manager`), or the slashless
handle (`tinyhat--agents--bot-manager`). Prefer the slashless handle
in HAPI URLs so canonical handles are not nested as path segments
inside another `/agents/...` route. They manage **two orthogonal
axes** for an agent:

- **Access mode** — the closed enum `restricted | account_members |
  whitelist | public_with_blacklist` that decides who can chat
  with the agent.
- **Access list** — a flat set of `(user_id, kind, is_admin)` rows
  where `kind` is `'allow'` (whitelist) or `'deny'` (blacklist) and
  `is_admin` carries the per-agent admin tier (orthogonal to
  `kind`; a user can be a chat-allowed admin, a chat-allowed
  non-admin, or a denied non-admin — but never a denied admin).

Every mutation requires the **caller** to be admin of the agent.
The agent's primary owner is implicit admin; tinyloop superadmins
are admin of every agent; anyone else needs an explicit
`is_admin=true` access-list entry. The `set-access-mode` skill is
the user-facing workflow that drives these operations.

Access-list mutations are also scoped to the target agent. The
`user_id` in `agents.access-list.entries.upsert` and `.delete` is
not a global-user escape hatch; the platform only accepts users who
are already in the target agent's scope (for example the owner, an
account member, someone who has talked to that agent, or an
existing row on that agent's access list).

**`X-Tinyhat-Acting-User` is server-injected.** When a per-agent
admin endpoint runs, the platform automatically attaches the
`X-Tinyhat-Acting-User` header from the chat run context (the
Telegram user the bot is processing a message for) before the
`gated_api_call` reaches the upstream endpoint. The agent does NOT
set this header from inside the sandbox — the gate strips
caller-supplied auth headers, and the acting-user header lives in
that same trusted-injection bucket. The per-agent admin check then
runs against the resolved acting user, NOT against the agent's
owner. So when alice DMs the bot-manager and asks to flip the
access mode, the platform evaluates "is alice admin?" — not "is
the bot-manager's owner admin?" — and refuses with 403 if she
isn't.

#### `agents.access-mode.get` — `GET /hapi/v1/agents/{ex_id_or_handle}/access-mode`

Read the current access mode. Response:
`{agent_id, agent_handle, mode, allowed_modes}`. The `allowed_modes`
list is the closed enum a UI / skill can render as a dropdown
without baking the spec into its prompt.

#### `agents.access-mode.set` — `POST /hapi/v1/agents/{ex_id_or_handle}/access-mode`

Set the agent's access mode. Body:
`{"mode": "restricted" | "account_members" | "whitelist" | "public_with_blacklist"}`.
Idempotent — the same mode twice returns the same state. 400 on
unknown enum value, 403 when caller is not admin.

#### `agents.access-list.get` — `GET /hapi/v1/agents/{ex_id_or_handle}/access-list`

List every `(agent, user)` access-list row for the agent.
Response:
`{agent_id, agent_handle, entries: [{agent_id, user_id, kind, is_admin, added_by_user_id}, …]}`.
Caller must be admin — the admin-tier flag is privileged metadata.

#### `agents.access-list.entries.upsert` — `POST /hapi/v1/agents/{ex_id_or_handle}/access-list/entries`

Add or update an `(agent, user)` row. Body:
`{"user_id": <int>, "kind": "allow" | "deny", "is_admin": bool}`.
Idempotent on `(agent_id, user_id)`. Refuses
`{"kind": "deny", "is_admin": true}` with 409 (a denied user can't
be admin). 404 when the target user_id doesn't exist. 403 when
caller isn't admin. 404 when the target user exists globally but is
outside this agent's scope.

#### `agents.access-list.entries.delete` — `DELETE /hapi/v1/agents/{ex_id_or_handle}/access-list/entries/{user_id}`

Remove an `(agent, user)` row. 404 when the row doesn't exist (the
caller can present this to the user as "no entry to remove" rather
than as an error).

### Handles & repos

#### `handles.check` — `GET /hapi/v1/handles/check`

Is `<handle>` available? Query: `scope=account|agent`,
`handle=<kebab-case>`, `account_handle=<slug>` (only for
`scope=agent`). Returns `{available: bool, reason: …}`.

#### `repos.list` — `GET /hapi/v1/repos`

List Tinyhat-managed repositories (platform-owned + per-agent
forks). Query: `scope=platform|account|all`, `kind=<…>`,
`account_handle=<slug>`, `status=…`, `limit`, `offset`.

### Admin / platform-bootstrap status

#### `admin.platform.account.status` — `GET /hapi/v1/admin/platform/account/status`

Read-only check for the platform `tinyhat_accounts` row. Returns
`{exists: bool, account_id, kind, slug, display_name, member_count}`.

#### `admin.platform.account.seed` — `POST /hapi/v1/admin/platform/account/seed`

Idempotently ensure the platform account exists.

#### `admin.platform.repos.status` — `GET /hapi/v1/admin/platform/repos/status`

Per-handle status snapshot for every repo the runtime helpers
expect (bot-manager seed, default-agent template, harness
mirrors).

#### `admin.platform.repos.seed` — `POST /hapi/v1/admin/platform/repos/seed`

Idempotently seed every missing platform repo.

#### `admin.platform.agent.status` — `GET /hapi/v1/admin/platform/agent/status`

Whether the platform `kind=system` agent (the bot-manager) row
exists yet. When `exists=false` the response carries
`create_url=/admin/that#create-platform-agent` so the maintainer
knows where to go.

### User-owned vaults (chat-first credential storage)

The chat-first surface for user-owned credentials (OpenAI keys,
GitHub PATs, Slack tokens, …). Every agent the user owns reads
from these rows via `gated_api_call`'s `owner_kind: user` ACL
entries; they replace the planned `agents.set_credential` flow,
which was per-agent and is dropped post-#209.

Vault rows are keyed on `(owner_kind='user', owner_id=<chatting
user>, vault_name=<bag>)`. The platform server-injects
`X-Tinyhat-User-Id` from the run context — you do **not** know or
set the header. Each row is a bag of `KEY=VALUE` entries; e.g.
`vault_name='openai'` with `OPENAI_API_KEY=sk-…`.

#### `users.me.vaults.list` — `GET /hapi/v1/users/me/vaults`

List the calling user's vault rows. Response carries
`{user_id, vaults: [{vault_name, key_version, keys: [{key,
has_value}]}]}`. **Never returns plaintext values** — only the
key shape — so the agent can confirm "is this set?" without
seeing secrets.

#### `users.me.vaults.upsert` — `POST /hapi/v1/users/me/vaults`

Add or update a key in one of the user's vaults. Body:
`{vault_name: "<bag>", env: {KEY: VALUE}}`. **Merge semantics:**
sibling keys in the same bag survive; an empty-string value
deletes that key (so the chat workflow can clear an entry without
an extra round-trip). Response is the post-write key shape; values
are never echoed.

#### `users.me.vaults.delete` — `DELETE /hapi/v1/users/me/vaults/{vault_name}`

Drop a whole vault row. Response confirms `{deleted: bool}`.

#### `users.me.vaults.delete_key` — `DELETE /hapi/v1/users/me/vaults/{vault_name}/keys/{key}`

Surgical removal of a single key from a vault row. The row stays
intact when other keys remain; it's deleted when the last key is
removed.

### OpenAPI

#### `spec.json` — `GET /hapi/v1/openapi.json`

The OpenAPI 3.1 document for the `/hapi/v1` surface, admin-gated
(same gate as every other call you make through the gate). Fetch
this when the user asks "what new endpoints landed?" — don't trust
your memory of this file over the live spec.

The companion **Swagger UI** lives at `GET /hapi/v1/docs` (also
admin-gated; `try-it-out` disabled for write methods). It is a
human-only surface — point a maintainer at it when they ask "where
do I poke at the platform API in a browser?", but never `gated_api_call`
it yourself; the response body is HTML, not JSON.

## Planned operations (citable now, will land per issue)

These operationIds are **not yet implemented** but are the names
the per-operation skills cite. When the user asks for one of these
and the endpoint isn't shipped yet, the per-operation skill will
tell them so plainly.

- `agents.get`, `agents.transfer`, `agents.suspend` — provisioning
  lifecycle (delete already shipped on hapi).
- `agents.set_harness`, `agents.set_model`, `agents.show_config` —
  per-agent config. Note: `agents.set_access_mode` graduated to
  the live ops `agents.access-mode.{get,set}` +
  `agents.access-list.{get,entries.upsert,entries.delete}` (above),
  and `agents.set_credential` is **not** on this list anymore —
  credentials are user-owned and live under `users.me.vaults.*`
  (above), not on the agent.
- `agents.list_models` — list models the platform currently
  supports for `agents.set_model`.
- `channels.register`, `channels.list`, `channels.unregister` — bind
  Telegram bots / Slack workspaces / etc. to an agent.
- `skills.add_from_handle`, `skills.extract`,
  `skills.refresh_vendored`, `skills.list_for_agent` — skill
  lifecycle.
- `hats.get`, `hats.customize_soul`, `hats.add_toolkit` — hat
  manifest edits.
- `accounts.add_member`, `accounts.list_members` — multi-tenant
  membership.
- `whitelist.list_pending` — outstanding invite requests (the
  invite-prompted-by-canned-reply flow). The grant / revoke
  primitives are now subsumed by
  `agents.access-list.entries.upsert` / `.delete` (above).
- `harnesses.versions.list` — pick a harness version per agent.
- `evals.run`, `evals.compare_models` — evaluation runs.
- `runs.list`, `runs.get`, `conversations.list`,
  `conversations.get`, `users.list`, `users.update` — observability
  + user management surfaces. Read-only equivalents may already
  exist on a maintainer-only legacy path; the agent does not call
  those.

## When to fetch the live spec instead of trusting this file

This SKILL.md is a frozen snapshot. When the user asks something
like "what new endpoints landed this month?" or "what shape does
the X endpoint accept?", **fetch `spec.json`** (the `/hapi/v1`
spec) **and read it**. Don't trust your memory of this file over
the spec.

## What to put in your final assistant message

You almost never speak to the user directly from this skill —
this is a reference, not a workflow. The per-operation skills
(`provision-user-agent`, `register-telegram-channel`, …) own the
user-facing replies. They cite this file by operationId; trust
their composition rules for the actual reply text.
