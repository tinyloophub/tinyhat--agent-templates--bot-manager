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
- `set-access-mode` "Identifying this agent on the URL" section
  spells out that the platform's resolver only accepts a numeric
  agent id or the canonical `<account-slug>/agents/<agent-name>`
  handle — placeholders like `<self>` would 404 — and that the
  bot-manager template uses `tinyhat/agents/bot-manager` literally
  on every URL because it is the platform's `is_platform: true`
  hat.

### Changed

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
