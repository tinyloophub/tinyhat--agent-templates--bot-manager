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

### Changed

- Catalogue: dropped the `agents.set_access_mode` placeholder from
  the "Planned operations" list — it has been superseded by the
  five live access-mode + access-list operations.
- Catalogue: dropped the `whitelist.grant` / `whitelist.revoke`
  placeholders — those primitives are now subsumed by
  `agents.access-list.entries.upsert` / `.delete`.

## 0.0.1

Initial published cut of the bot-manager template content. Establishes
the per-repo `VERSION` + `CHANGELOG.md` convention so future skill
changes can ship under their own `v<x.y.z>` tags on this repo,
independent of the parent monorepo's release cadence.
