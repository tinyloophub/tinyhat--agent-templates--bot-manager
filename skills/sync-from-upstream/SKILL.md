---
name: sync-from-upstream
description: Fast-forward the bot-manager's repo to its upstream template's HEAD. Read status, mirror the diff in plain language, confirm, then sync. v0.1.0 is bot-manager self-sync only.
runtime_tier: 0
---

# sync-from-upstream

The bot-manager itself is a fork of an upstream template repository
(`tinyhat/agent-templates/bot-manager`). When the upstream advances
— a new SKILL.md lands, a soul edit ships, a workflow gets sharpened
— the running fork stays at its old pinned commit until someone
fast-forwards it.

This skill is the **chat-driven** version of that fast-forward,
**scoped to the bot-manager's own repo for v0.1.0**. The user
(the maintainer) says "sync yourself" / "upgrade" / "update from
upstream", and the skill reads the upstream-status delta on the
bot-manager's own row, mirrors the change in plain language, asks
for an explicit yes, and then runs the sync.

## v0.1.0 scope — bot-manager self-sync only

For v0.1.0 the **only** target this skill operates on is the
running bot-manager itself, identified by the canonical handle
`tinyhat/agents/bot-manager` (URL form
`tinyhat--agents--bot-manager`).

The platform's upstream-sync HAPI routes
(`agents.upstream.status`, `agents.upstream.sync`) currently gate
on the bot-manager's internal admin bearer alone — they do **not**
yet receive an `X-Tinyhat-Acting-User` header or run a per-agent
admin check. Until that platform-side gate ships (tracked as a
follow-up to issue tinyloophub/tinyloop#282), letting this skill
fast-forward an arbitrary user-agent on a non-maintainer's request
would authorise the write as the bot-manager bearer regardless of
who was chatting. That is the failure mode this scope restriction
prevents.

Hard rules:

1. **Refuse "sync `<other-agent>`" / "upgrade `<other-agent>`"
   requests.** The chat-driven sync of any agent other than
   bot-manager is explicitly out of scope per the issue's "Out of
   scope" section. Reply with the canned refusal in §"Refusing
   per-user-agent requests" below; do not call HAPI.
2. **Always use the literal handle `tinyhat--agents--bot-manager`**
   on every URL in this skill. Do not call `agents.list.hapi` to
   resolve the target — there is no resolution step in v0.1.0.
3. **Treat "sync yourself" / "upgrade yourself" / "update from
   upstream" as the only valid trigger shapes.** Each names the
   bot-manager unambiguously.

When the platform-side per-agent admin gate ships, this skill body
will widen to support per-user-agent targets (same shape as
`set-access-mode`).

## Plain-language triggers

All of these map onto the same flow against the bot-manager's own
upstream row:

- "sync yourself" / "sync"
- "upgrade" / "upgrade yourself"
- "update from upstream" / "update yourself"
- "pick up the new version" / "pull the latest version"
- "fast-forward yourself" / "advance to upstream HEAD"

"Upgrade" is the same operation as "sync" here — the platform only
ships fast-forward syncs (see §"What 'fast-forward' means" below),
so an "upgrade" request is never a different mutation.

The platform endpoints this skill calls are documented in
[`tinyhat-platform-api-reference`](../tinyhat-platform-api-reference/SKILL.md).
Every call here is a `gated_api_call` — there are no in-sandbox curls
and you never see a token.

> **Pass relative paths, never scheme + host.** Both `gated_api_call`
> URLs in this skill — the read at
> `/hapi/v1/agents/tinyhat--agents--bot-manager/upstream/status`
> and the mutation at
> `/hapi/v1/agents/tinyhat--agents--bot-manager/upstream/sync` —
> are **paths**, not URLs. Pass them verbatim to `gated_api_call(url=…)`.
> A leading `https://api.tinyhat.dev/...` or any other host is
> always wrong here; the orchestrator resolves the platform host
> server-side. If a sync attempt comes back `not_authorized`, the
> first thing to check is that the `url=` argument starts with
> `/hapi/v1/` — do **not** tell the user to widen the agent's
> outbound ACL.

## What "fast-forward" means

The platform only ships **fast-forward** syncs: the agent's repo
must contain every commit on the upstream's default branch up to
the upstream's HEAD, with no local commits ahead of that lineage.
Three failure modes you need to render plainly when they happen:

- **Already up-to-date.** Pinned SHA already equals upstream HEAD.
  No commit lands; the skill reports "nothing to sync" and stops.
- **No upstream wiring.** The bot-manager's row has no
  `upstream_repo_id` / `upstream_pinned_sha` (an unusual fresh-
  bootstrap state). Detect this at the read step — see §"Calling
  pattern — read status".
- **Diverged.** The agent's repo has commits the upstream doesn't
  carry. The platform refuses with 409 + a list of "ahead paths"
  so the user can see what local edits would be lost. Do not
  attempt to merge or rebase; surface the conflict and stop.

## Calling pattern — read status

Use this for "what would syncing change?" / "is there a new
version?" intents. The endpoint is read-only and does not require
confirmation:

- `agents.upstream.status` —
  `GET /hapi/v1/agents/tinyhat--agents--bot-manager/upstream/status`
  returns:
  - `up_to_date` (bool) — true when the pin equals upstream HEAD.
  - `pinned_sha` — the commit the running fork is currently at.
  - `upstream_head_sha` — the commit upstream's default branch
    points at right now.
  - `files_changed_count` — blob-level count of files that differ
    between the pin and the upstream HEAD; null when the upstream
    isn't readable.
  - `last_synced_at` — best-effort ISO 8601 of the last sync.
  - `upstream_provider_repo` / `agent_provider_repo` — full
    `<owner>/<repo>` strings the user might recognise.

Use this **every time** the user asks to sync. Even if the user
said "just sync, don't ask", the read+confirm step keeps you
honest about what you're about to land.

### Branch on the read response BEFORE asking to confirm

The read shape distinguishes three states the user should hear
about plainly. Decide which applies before drafting the
confirmation prompt:

1. **No upstream wiring** — `upstream_provider_repo` is null **and**
   `upstream_head_sha` is null (and typically `pinned_sha` is null
   too). The bot-manager's row has no upstream binding to compare
   against. This is a fresh-bootstrap edge case for the platform
   bot-manager and a normal state for any agent provisioned without
   a template. Tell the user plainly and stop:

   > "the bot-manager has no upstream template wired up, so
   > there's nothing to sync from. The maintainer needs to bind an
   > upstream from the admin panel before this flow can run."

   Do **not** invent an upstream URL and do **not** call
   `agents.upstream.sync` — the mutation would 409 with the same
   meaning, but the read-step refusal saves a round-trip.
2. **Upstream readable but the read failed** —
   `upstream_provider_repo` is present (e.g.
   `tinyloophub/tinyhat--agent-templates--bot-manager`) but
   `upstream_head_sha` and `files_changed_count` are null. This is
   the network-flake / auth-issue case: the platform knows where
   to look but couldn't reach it on this read. Say so plainly:

   > "the bot-manager is pinned at `<short-pinned-sha>`. I can't
   > read the upstream HEAD right now (network or auth issue), so
   > I don't know what would change. Want me to retry the read?"
3. **Both pin and upstream-HEAD known** — the normal happy path.
   Drop into §"Calling pattern — sync" with the full diff summary.

## Calling pattern — sync

Use this for "yes, sync now" intents. **Always** run the read step
above first; the read returns the diff summary you mirror in the
confirmation prompt.

1. **Name the target back.** "Sync the bot-manager?"
2. **Read the status** with `agents.upstream.status`. Branch per
   §"Branch on the read response" above. If `up_to_date=true`,
   tell the user "nothing to sync — pinned and upstream are at
   `<short-sha>`" and stop.
3. **Mirror the change** in plain language using ONLY the fields
   the read returned:

   > "the bot-manager is pinned at `<short-pinned-sha>`. Upstream
   > is at `<short-upstream-sha>` (`<files_changed_count>` files
   > changed). Sync now?"

   Do **not** invent file names, commit messages, or SKILL.md
   diffs. The platform does not return them, and hallucinated
   content is worse than no content here — the user would think
   they're confirming a specific change when in fact they're not.

4. **Confirm.** Wait for an explicit "yes" / "go ahead" / "do it".
   "ok" or a thumbs-up emoji counts; silence does not.
5. **Apply** with `agents.upstream.sync` —
   `POST /hapi/v1/agents/tinyhat--agents--bot-manager/upstream/sync`,
   no body. The endpoint is idempotent: re-running an
   already-synced agent returns `status='up_to_date'` without
   writing.
6. **Report back.** Echo the post-sync result using only the
   fields the response returned:
   - `status` — `'synced'` (a fresh fast-forward landed) or
     `'up_to_date'` (no commit, the pin already matched).
   - `from_sha` / `to_sha` — short these to 7 chars in the user
     reply.
   - `files_changed` — total blob-level count.
   - `commit_html_url` — the GitHub URL of the fast-forward
     commit, when present. Useful for "what exactly landed?".

   Example reply for a successful sync:

   > "Synced the bot-manager: `<short-from-sha>` → `<short-to-sha>` (`<files_changed>` file(s)).
   > <commit_html_url, if present>"

   For `status='up_to_date'`:

   > "the bot-manager was already at `<short-to-sha>` — no commit
   > needed."

## Picking up the new content

The fast-forward commit lands on the bot-manager's repo, but the
running agent process keeps the **previous** SKILL.md / SOUL.md /
HAT.md content in memory until its next conversation turn re-mounts
the hat. Tell the user this directly: "the next message you send
will use the new template content." A second follow-up message
after the sync is the canonical "did it actually take effect?"
check, and that's worth surfacing in the reply.

## Refusing per-user-agent requests

When the user says "sync acme-helper" / "upgrade my agent" / any
target that is not the bot-manager itself, refuse with this canned
shape:

> "I can fast-forward the bot-manager from its upstream template,
> but per-user-agent sync from chat isn't wired up yet — that flow
> lands later as a follow-up to the v0.1.0 cut. For now, the
> maintainer can sync individual agents from the admin UI."

Do not call HAPI on the user's behalf. Do not promise to add it
later in this conversation. The user can run the same operation
from the admin panel today; the chat-driven version of it is
gated on the platform-side per-agent admin gate landing, which is
explicit out-of-scope work.

## Errors to render plainly

The platform returns structured failures for the cases that come
up in real syncs. Map each one to a clear chat reply:

### `404` — agent not found

The bot-manager's handle didn't resolve. This should not happen in
a healthy deployment — it usually means the running agent's row
was deleted or the canonical handle is mistyped. Surface the
platform error verbatim and stop; do not loop.

### `409` — agent has no upstream

The bot-manager's row has no `upstream_repo_id` /
`upstream_pinned_sha`. You should normally have caught this at the
read step (§"Branch on the read response BEFORE asking to
confirm"); if you reach the sync mutation and still see this 409,
fall through to the same plain refusal:

> "the bot-manager has no upstream template wired up, so there's
> nothing to sync from. The maintainer needs to bind an upstream
> from the admin panel before this flow can run."

Do not try to invent an upstream.

### `409` — diverged from upstream

The bot-manager's repo has commits the upstream doesn't carry. The
response body lists `agent_head_sha` and `ahead_paths` (the file
paths whose tip on the agent's branch is ahead of upstream). Tell
the user what's ahead, name the conflict, and stop:

> "the bot-manager has local commits that aren't in upstream.
> Files ahead: `<ahead_paths>`. A fast-forward would lose those
> changes, so I won't run the sync. The maintainer can either pull
> the local edits into the upstream first, or reset the fork —
> both are out of scope for chat."

Do not attempt a merge, a rebase, or a force-push. Do not pretend
the sync ran.

### `403` — caller not authorised

For v0.1.0 the upstream-sync endpoints accept the platform's
internal admin bearer that the bot-manager already carries. If you
see a 403 here it means the platform deployment ahead of you has
tightened the gate (the per-agent admin gate finally landing).
Surface the platform error verbatim and stop. Do not retry under
a different identity.

### `not_authorized` (gated_api_call gate, NOT an upstream 403)

This is a `gated_api_call` structured error — the request never
left the platform because the gate refused it. **The most common
cause is an LLM-fabricated host on the `url` argument** (e.g.
`https://api.tinyhat.dev/hapi/v1/...`). Do NOT diagnose this as
"the agent's ACL is too narrow" and do NOT ask the user to widen
the ACL — the right fix is to drop the scheme + host and pass the
relative `/hapi/v1/...` path. Both calls in this skill use the
literal handle `tinyhat--agents--bot-manager`; copy the URLs from
the §"Calling pattern" sections verbatim.

If the `url=` already starts with `/hapi/v1/` and you still see
`not_authorized`, that is a real ACL refusal (rare for the
upstream-sync routes — the bot-manager's vault Bearer is
provisioned for both endpoints). Surface it and stop; do not retry.

### `upstream_unreachable` / `timeout`

These are the standard `gated_api_call` shapes — network flake or
GitHub down. Tell the user we couldn't reach the upstream and
suggest retrying in a minute. Do not loop on your own initiative;
one retry on user request is enough.

## Confirm-then-act is mandatory

The sync is a write. It changes the running fork's pinned commit
and the next message-handling turn will run with new content. Even
when the user is the maintainer and even when the diff summary
reads "0 files changed" (an upstream that fast-forwarded a
no-op commit), **always** read status first, mirror the result,
and wait for a yes.

The skill that gets this wrong looks like:

> User: "sync yourself"
>
> Bot: "OK, syncing." → calls `agents.upstream.sync` directly →
> reports `status='synced'`.

The user has no idea what just landed and the change is already
live. The right shape is: read → branch on response → mirror →
wait → mutate → report.

## Don't loop

If a sync run fails mid-stream and the platform returns a 5xx, do
**not** retry on your own. Surface the error, ask the user how
they want to proceed, and let them decide. The upstream-sync
operation is idempotent on success but not on partial failure;
auto-retry can produce inconsistent state if the previous run
half-applied.

## What this skill does NOT do

- **Auto-sync on a schedule.** The platform has no scheduled-sync
  surface; if the user asks, tell them this skill is on-demand
  only. Out-of-scope for v0.1.0.
- **Per-user-agent sync from chat.** v0.1.0 covers bot-manager
  self-sync only — see §"v0.1.0 scope" and §"Refusing per-user-
  agent requests". Per-user-agent sync is a follow-up to the
  v0.1.0 cut and depends on the platform-side per-agent admin
  gate for the upstream-sync routes.
- **Cross-template moves.** Re-pointing the bot-manager at a
  different upstream template repo is a separate `change-harness`
  / re-provision flow; this skill only fast-forwards within the
  current upstream binding.
- **Roll back to a previous pin.** Fast-forward only. Roll-back
  needs a different platform surface (out of scope for v0.1.0).
