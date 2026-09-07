# Changelog

This repository follows its own release cadence (independent of any
parent monorepo gitlinks that point at it). Each release ships as a
`v<x.y.z>` tag on this repo.

## Unreleased

_(Empty.)_

## 0.0.2 — 2026-09-07

Publish the template content already present on main so consumers can pin a
released artifact instead of an untagged commit. This cut changes release
metadata only relative to the existing main payload.

- Reconcile the populated HAT, platform API reference, provisioning, channel,
  access-mode and credential skills with their smoke cases.
- Include the public release naming and marker runbook.


## 0.0.1 — 2026-05-07

First published cut. Establishes the per-repo `VERSION` + `CHANGELOG.md`
convention so future skill changes ship under their own `v<x.y.z>` tags
on this repo, independent of the parent monorepo's release cadence.
Lands in lockstep with the tinyloop monorepo's `v0.1.0` cut, providing
the chat-driven skill bodies for M0.1.0-B (mode toggle), M0.1.0-C
(upstream sync), and M0.1.0-D (Managed Bots user-agent provisioning).

### Added

- `provision-user-agent` skill body now drives the Telegram Managed
  Bots zero-token flow: distinguish Tinyhat account/agent handles
  from Telegram usernames, generate Telegram-safe username
  candidates, confirm the plan, call `propose_managed_bot_creation`,
  send the `t.me/newbot` link, retry with fresh candidates if
  Telegram reports the username unavailable, wait for a matched
  `managed_bot` confirmation, then call `agents.create(kind='user')`
  with `managed_bot_id` and without `telegram_bot_token`.
- `tinyhat-platform-api-reference` documents the
  `propose_managed_bot_creation` function tool, Telegram username
  rules, the fact that Telegram's confirmation sheet is authoritative
  for global availability, matched-vs-unmatched managed-bot
  confirmations, user-agent `agents.create` request shape, creator
  binding, persisted non-secret bot metadata, and restricted
  creator-admin access defaults.
- `set-access-mode` skill body now drives the four canonical platform
  access modes (`restricted`, `account_members`, `whitelist`,
  `public_with_blacklist`) plus the per-agent admin tier through
  the live `agents.access-mode.{get,set}` and
  `agents.access-list.{get,entries.upsert,entries.delete}`
  operationIds. Replaces the earlier "endpoint not yet shipped"
  body wired against the planned `agents.set_access_mode`.
- `tinyhat-platform-api-reference` catalogue documents the same five
  operationIds and notes that `X-Tinyhat-Acting-User` is
  server-injected by the platform's `gated_api_call` gate (the
  agent must NOT set the header from inside the sandbox).
- `set-access-mode` "Identifying the target agent on the URL"
  section spells out the three identifier shapes the platform's
  resolver accepts — a numeric `tinyhat_agents.id`, the canonical
  `<account-slug>/agents/<agent-name>` handle, or the slashless
  `<account-slug>--agents--<agent-name>` form (preferred in HAPI
  URLs because it does not embed a three-segment handle inside
  another `/agents/...` route). Placeholders like `<self>` /
  `<this>` / `me` / `{agent_identifier}` 404. The skill resolves
  the target the user named first; `tinyhat/agents/bot-manager`
  is only used when the user explicitly targets the platform
  bot-manager agent.
- `sync-from-upstream` skill body — chat-driven fast-forward of
  **the bot-manager's own repo** to the latest commit on its
  upstream template, via the live `agents.upstream.{status,sync}`
  operationIds. Recognises "sync yourself", "upgrade", "update
  from upstream", and "pick up the new version" as triggers (all
  map onto the same fast-forward operation). Reads the status
  delta first and branches on the response shape — no-upstream
  wiring (`upstream_provider_repo` and `upstream_head_sha` both
  null) is detected at read time and refused plainly without
  burning a sync round-trip; upstream-unreadable
  (`upstream_provider_repo` present but the head SHA null) prompts
  the user to retry the read; the normal happy path mirrors the
  diff in plain language using only the platform-returned fields
  (no invented SKILL.md content), waits for an explicit yes, then
  syncs and reports the new pinned commit. v0.1.0 is **bot-manager
  self-sync only**; per-user-agent sync from chat is a follow-up
  that depends on the platform-side per-agent admin gate for
  `agents.upstream.{status,sync}` landing first.

### Changed

- `register-telegram-channel` is now explicitly scoped to a future
  import-existing-bot escape hatch. Managed Bots agents created by
  `provision-user-agent` are bound automatically during
  `agents.create`, so the skill no longer appears as the second half
  of new user-agent provisioning.
- `HAT.md` lists the `propose_managed_bot_creation` backend function
  tool and describes `provision-user-agent` as the Managed Bots
  workflow, including creator-only initial access and later sharing
  through access-mode/access-list operations.
- Smoke evals cover the zero-token Managed Bots create path and the
  unavailable-username retry path, plus the register-channel refusal
  for managed-bot agents.
- `tinyhat-platform-api-reference`, `sync-from-upstream`,
  `set-access-mode`, and `set-credential` skill bodies sharpen the
  "pass a relative `/hapi/v1/...` path" guidance into an explicit
  anti-pattern callout that names the LLM-fabricated hosts the
  agent must never prepend (`api.tinyhat.dev`, `api.tinyloop.co`,
  `<account>.tinyhat.dev`, `localhost:...`). The
  `tinyhat-platform-api-reference` calling-pattern template now
  includes a wrong-vs-right URL table, and the `not_authorized`
  error row tells the agent to check the URL shape before
  diagnosing an ACL refusal — fixing the LLM-hallucination root
  cause described in the issue, not asking the user to widen the
  agent's outbound ACL for a fabricated hostname.
- `sync-from-upstream` and `set-access-mode` add a
  `not_authorized` (gated_api_call gate) error section that
  distinguishes the gate refusal (typically a fabricated host on
  the `url=` argument) from the per-agent admin / upstream `403`,
  so the agent renders the right diagnosis to the user.
- `set-access-mode` now treats bot-manager as only one possible
  target agent. It resolves the agent the user named and calls the
  generic `/hapi/v1/agents/{ex_id_or_handle:path}/...` routes with
  that agent's numeric id or slashless handle, rather than hard-
  coding the platform bot-manager handle into every call. Slash-
  bearing canonical handles still work, but slashless identifiers
  keep HAPI URLs readable.
- `set-access-mode` now states the per-agent admin and target-user
  scoping rules explicitly: the chatting user must be admin of the
  target agent, and access-list mutations may only use user ids that
  came from agent-scoped context.
- Catalogue: dropped the `agents.set_access_mode` placeholder from
  the "Planned operations" list — it has been superseded by the
  five live access-mode + access-list operations.
- Catalogue: dropped the `whitelist.grant` / `whitelist.revoke`
  placeholders — those primitives are now subsumed by
  `agents.access-list.entries.upsert` / `.delete`.
- `HAT.md` workflow-skills summary: `set-access-mode` is now
  described by the four canonical mode names + the access-list
  + per-agent admin tier, not the old `public / invite_only /
  private` legacy names. The "Several depend on platform endpoints
  that haven't shipped yet" preamble is softened to "Some depend"
  because `set-access-mode` is now live.
- `show-config` skill body: agents.list `access_mode` summary now
  cites the four canonical platform values and points the agent at
  `agents.access-list.get` for "who's whitelisted / blacklisted /
  admin" lookups, instead of the legacy `public / invite_only /
  private` triple.

