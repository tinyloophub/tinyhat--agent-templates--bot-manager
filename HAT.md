---
name: bot-manager
description: The @tinyhatbot Telegram manager bot — the front desk that meets every new user.
soul: SOUL.md
is_platform: true
toolkits:
  - bot-manager-20260427
  - outbound-20260427
skills:
  - name: reply-via-telegram
    source: local
    path: skills/reply-via-telegram
  - name: tell-joke
    source: local
    path: skills/tell-joke
  - name: tinyhat-platform-api-reference
    source: local
    path: skills/tinyhat-platform-api-reference
  - name: provision-user-agent
    source: local
    path: skills/provision-user-agent
  - name: register-telegram-channel
    source: local
    path: skills/register-telegram-channel
  - name: customize-soul
    source: local
    path: skills/customize-soul
  - name: add-skill-from-repo
    source: local
    path: skills/add-skill-from-repo
  - name: extract-skill
    source: local
    path: skills/extract-skill
  - name: set-access-mode
    source: local
    path: skills/set-access-mode
  - name: sync-from-upstream
    source: local
    path: skills/sync-from-upstream
  - name: change-model
    source: local
    path: skills/change-model
  - name: change-harness
    source: local
    path: skills/change-harness
  - name: set-credential
    source: local
    path: skills/set-credential
  - name: show-config
    source: local
    path: skills/show-config
---

# bot-manager

This is the manifest for the `@tinyhatbot` manager hat.

The frontmatter above is the machine-readable part — it's what
`skill_loader_service` (and, eventually, the manifest resolver) reads
to decide which skills and function toolkits to mount. Everything
below the frontmatter is prose for the maintainer.

## Skills

The manager hat carries a set of **sandbox skills** (mounted as
files under `/skills/<name>/` inside the OpenAI hosted shell) plus
two **function toolkits** (`bot-manager-20260427` and
`outbound-20260427`, declared in `toolkits:` above) whose tools
run in our backend process.

Composition skills (always-on guidance):

- **reply-via-telegram** — *guidance file.* Tells the agent how to
  compose Telegram-shaped text and points at the
  `reply_via_telegram` function tool to actually send. The send is
  the function tool's job; this SKILL.md is just a how-to. See
  [skills/reply-via-telegram/SKILL.md](skills/reply-via-telegram/SKILL.md).
- **tell-joke** — sample skill demonstrating the SKILL.md format
  and the "compose a reply, then call `reply_via_telegram`"
  pattern.

Platform-API reference (cited by every workflow skill below):

- **tinyhat-platform-api-reference** — manpage of every Tinyhat
  platform admin endpoint the bot-manager can reach via
  `gated_api_call`. Cite by `operationId`, never URL.
  ([issue #102](https://github.com/tinyloophub/tinyloop/issues/102))

Workflow skills — each one is a short conversation pattern for one
maintenance operation, citing operations by id from the reference
above. Some depend on platform endpoints that haven't shipped yet;
those skill bodies tell the user "not yet wired up" rather than
inventing endpoints:

- **provision-user-agent** — create a new agent for a user.
- **register-telegram-channel** — bind a Telegram bot to an agent.
- **customize-soul** — edit a hat's identity prose.
- **add-skill-from-repo** — mount an external skill on an agent.
- **extract-skill** — publish an in-tree skill as its own repo.
- **set-access-mode** — manage the access mode (`restricted` /
  `account_members` / `whitelist` / `public_with_blacklist`) and
  the per-agent access list (allow / deny + admin tier). Live.
- **sync-from-upstream** — fast-forward the bot-manager's own
  repo to the latest commit on its upstream template. Triggered
  by "sync yourself", "upgrade", "update from upstream", or "pick
  up the new version" intents (all the same fast-forward
  operation). Read status, branch on the response shape (no-
  upstream / upstream-unreadable / normal), mirror the diff in
  plain language, ask for explicit confirmation, then sync. Live
  for **bot-manager self-sync only** in v0.1.0; per-user-agent
  sync from chat is a follow-up that depends on the platform
  per-agent admin gate for the upstream-sync routes.
- **change-model** — swap the LLM an agent runs against.
- **change-harness** — swap the runtime harness vendoring.
- **set-credential** — store an API key / token in the vault.
- **show-config** — read-only summary of an agent's config.

Function tools (from `bot-manager-20260427`):

- **`reply_via_telegram(text)`** — primary delivery path. Replaces
  the v0 in-sandbox curl which went through OpenAI's broken proxy
  on :8080 ([issue #62](https://github.com/tinyloophub/tinyloop/issues/62)).
- **`read_older_history(count, keyword, before_seq)`** — fetch
  older turns from the conversation bound to this run. Replaces the
  v0 `read-older-history` SKILL.md (same broken-proxy failure
  mode). Reads directly from `tinyhat_messages` in our backend; no
  sandbox network involved.

Function tools (from `outbound-20260427` — [issue #101](https://github.com/tinyloophub/tinyloop/issues/101)):

- **`gated_api_call(method, url, headers, json_body, query, timeout_seconds)`**
  — outbound HTTP gate. The single function tool every external
  HTTP from this agent goes through. Per-agent ACL match → vault
  credential injection → upstream HTTP → OTel-shaped audit row.
  Strips caller-supplied auth headers. Returns structured errors
  (`not_authorized`, `credential_missing`, `rate_limited`,
  `timeout`, `upstream_unreachable`) or the upstream response
  (`{status, headers, body, latency_ms}`). Pre-#104 — when no
  `tinyhat_agents` row is bound to this hat — every URL returns
  `not_authorized` (agent identity is the source of the ACL; no
  identity = no capability).

When more capabilities land (greet, help, register-skill, …),
prefer adding them as **function tools in the toolkit** rather than
in-sandbox skills — the function-tool path is the post-#68 default
because it doesn't depend on the sandbox proxy being healthy.

## Why `HAT.md`, not `HAT.yaml`

We standardise hats and skills on Markdown. The YAML frontmatter
gives parsers everything they need; the body lets the human (and the
agent itself, if we ever feed manifests into prompts) read what the
hat *is*. This matches the Markdown-first convention adopted by
SKILL.md, SOUL.md, and the rest of the agent ecosystem (Anthropic
skills, Codex prompts, Cursor rules).
