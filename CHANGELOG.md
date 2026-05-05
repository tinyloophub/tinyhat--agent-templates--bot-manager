# Changelog

This repository follows its own release cadence (independent of any
parent monorepo gitlinks that point at it). Each release ships as a
`v<x.y.z>` tag on this repo.

## Unreleased

### Added

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

### Changed

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

## 0.0.1

Initial published cut of the bot-manager template content. Establishes
the per-repo `VERSION` + `CHANGELOG.md` convention so future skill
changes can ship under their own `v<x.y.z>` tags on this repo,
independent of the parent monorepo's release cadence.
