---
name: sync-from-upstream
description: Fast-forward a target agent's repo to its upstream template's HEAD. Read status, mirror the diff in plain language, confirm, then sync. Never invent diff content.
runtime_tier: 0
---

# sync-from-upstream

Each running agent on the platform is a fork of an upstream
template repository (the bot-manager itself forks
`tinyhat/agent-templates/bot-manager`; user agents fork the
`default-agent` template, or whichever template they were created
from). When the upstream advances — a new SKILL.md lands, a soul
edit ships, a workflow gets sharpened — the running fork stays at
its old pinned commit until someone fast-forwards it.

This skill is the **chat-driven** version of that fast-forward.
The user says "sync yourself" / "update from upstream" / "pick up
the new version", and the skill resolves the target agent, reads
the upstream-status delta, mirrors the change in plain language,
asks for an explicit yes, and then runs the sync.

The platform endpoints this skill calls are documented in
[`tinyhat-platform-api-reference`](../tinyhat-platform-api-reference/SKILL.md).
Every call here is a `gated_api_call` — there are no in-sandbox curls
and you never see a token.

## Identifying the target agent on the URL

Every endpoint below is path-parameterised on the **target agent
identifier**. The platform's resolver accepts three shapes:

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
   the user named. If there is more than one match, ask which
   agent they meant.
3. If the user says "this bot", "you", "yourself", or
   "bot-manager", clarify whether they mean the manager bot
   itself or one of their agents. Only use
   `tinyhat/agents/bot-manager` (or the slashless
   `tinyhat--agents--bot-manager`) when the user explicitly chose
   the platform bot-manager agent.

After resolution, replace the route variable with the resolved
identifier. Prefer the slashless handle when you have one. Example:
for the target handle `tinyloop/agents/acme-helper`, the
upstream-status URL is
`/hapi/v1/agents/tinyloop--agents--acme-helper/upstream/status`.

## What "fast-forward" means

The platform only ships **fast-forward** syncs: the agent's repo
must contain every commit on the upstream's default branch up to
the upstream's HEAD, with no local commits ahead of that lineage.
Two failure modes you need to render plainly when they happen:

- **Already up-to-date.** Pinned SHA already equals upstream HEAD.
  No commit lands; the skill reports "nothing to sync" and stops.
- **Diverged.** The agent's repo has commits the upstream doesn't
  carry. The platform refuses with 409 + a list of "ahead paths"
  so the user can see what local edits would be lost. Do not
  attempt to merge or rebase; surface the conflict and stop.

## Calling pattern — read status

Use this for "what would syncing change?" / "is there a new
version?" intents. The endpoint is read-only and does not require
confirmation:

- `agents.upstream.status` —
  `GET /hapi/v1/agents/{resolved_agent_identifier}/upstream/status`
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

  Replace `{resolved_agent_identifier}` literally; do not send the
  brace-wrapped placeholder.

## Calling pattern — sync

Use this for "yes, sync now" intents. **Always** run the read step
above first; the read returns the diff summary you mirror in the
confirmation prompt.

1. **Resolve the target agent** (per the section above) and name it
   back. "Sync `<agent-handle>`?"
2. **Read the status** with `agents.upstream.status`. If
   `up_to_date=true`, tell the user "nothing to sync — pinned and
   upstream are at `<short-sha>`" and stop.
3. **Mirror the change** in plain language using ONLY the fields
   the read returned. Two correct shapes:

   > "`<agent-handle>` is pinned at `<short-pinned-sha>`. Upstream
   > is at `<short-upstream-sha>` (`<files_changed_count>` files
   > changed). Sync now?"

   When `files_changed_count` is null (upstream unreadable),
   say so plainly:

   > "`<agent-handle>` is pinned at `<short-pinned-sha>`. I can't
   > read the upstream HEAD right now (network or auth issue), so
   > I don't know what would change. Want me to retry the read?"

   Do **not** invent file names, commit messages, or SKILL.md
   diffs. The platform does not return them, and hallucinated
   content is worse than no content here — the user would think
   they're confirming a specific change when in fact they're not.

4. **Confirm.** Wait for an explicit "yes" / "go ahead" / "do it".
   "ok" or a thumbs-up emoji counts; silence does not.
5. **Apply** with `agents.upstream.sync` —
   `POST /hapi/v1/agents/{resolved_agent_identifier}/upstream/sync`,
   no body. Replace `{resolved_agent_identifier}` first. The
   endpoint is idempotent: re-running an already-synced agent
   returns `status='up_to_date'` without writing.
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

   > "Synced `<agent-handle>`: `<short-from-sha>` → `<short-to-sha>` (`<files_changed>` file(s)).
   > <commit_html_url, if present>"

   For `status='up_to_date'`:

   > "`<agent-handle>` was already at `<short-to-sha>` — no commit
   > needed."

## Picking up the new content

The fast-forward commit lands on the agent's repo, but the running
agent process keeps the **previous** SKILL.md / SOUL.md / HAT.md
content in memory until its next conversation turn re-mounts the
hat. Tell the user this directly: "the next message you send to
`<agent-handle>` will use the new template content." A second
follow-up message after the sync is the canonical "did it actually
take effect?" check, and that's worth surfacing in the reply.

## Errors to render plainly

The platform returns structured failures for the cases that come
up in real syncs. Map each one to a clear chat reply:

### `404` — agent not found

The resolver couldn't match the identifier you sent. Either the
target agent doesn't exist, or you used a placeholder
(`<self>` / `me` / a brace-wrapped variable) instead of resolving
first. Re-resolve from the user's request and try again; do not
loop.

### `409` — agent has no upstream

The target agent has no `upstream_repo_id` / `upstream_pinned_sha`
on its row — usually a custom agent that was provisioned without a
template, or one whose template binding was cleared. Tell the user
plainly: "this agent was set up without an upstream template, so
there's nothing to sync from." Do not try to invent an upstream.

### `409` — diverged from upstream

The running fork has commits the upstream doesn't carry. The
response body lists `agent_head_sha` and `ahead_paths` (the file
paths whose tip on the agent's branch is ahead of upstream). Tell
the user what's ahead, name the conflict, and stop:

> "`<agent-handle>` has local commits that aren't in upstream.
> Files ahead: `<ahead_paths>`. A fast-forward would lose those
> changes, so I won't run the sync. The owner can either pull the
> local edits into the upstream first, or reset the fork — both
> are out of scope for chat."

Do not attempt a merge, a rebase, or a force-push. Do not pretend
the sync ran.

### `403` — caller not authorised

For v0.1.0 the upstream-sync endpoints accept the platform's
internal admin bearer (the bot-manager's outbound credential),
not a per-chatter principal. If you ever see a 403 here it means
the platform deployment ahead of you has tightened the gate.
Surface the platform error verbatim and stop; the maintainer's
follow-up is to widen the per-agent admin gate per
`tinyloophub/tinyloop` issue threads about per-user-agent sync
from chat.

### `upstream_unreachable` / `timeout`

These are the standard `gated_api_call` shapes — network flake or
GitHub down. Tell the user we couldn't reach the upstream and
suggest retrying in a minute. Do not loop on its own initiative;
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
live. The right shape is: read → mirror → wait → mutate → report.

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
- **Per-user-agent sync from a non-admin chatter.** v0.1.0 wires
  the chat path for the maintainer (the platform's
  `is_tinyloop_admin` user) only. A non-admin chatter who DMs the
  bot-manager and asks to sync a different agent will hit the
  platform's per-agent admin gate elsewhere in the request chain
  before reaching this skill.
- **Cross-template moves.** Re-pointing a running fork at a
  different upstream template repo is a separate `change-harness`
  / re-provision flow; this skill only fast-forwards within the
  current upstream binding.
- **Roll back to a previous pin.** Fast-forward only. Roll-back
  needs a different platform surface (out of scope for v0.1.0).
