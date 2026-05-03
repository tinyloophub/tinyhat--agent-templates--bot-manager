# bot-manager — agent repo

This repository is the **bot-manager** agent's content: the
`HAT.md` manifest, the `SOUL.md` identity prose, every
`SKILL.md` under `skills/`, and the per-skill smoke evals
under `evals/`. The Tinyhat platform reads this tree at agent
mount time; the agent runs against the platform's harness with
this content as its system context.

The repo is **published**: every file you see here ships into
the agent's prompt verbatim. Do not put platform-internal paths,
secrets, or development-only references in any file inside this
tree — write everything as if the repo were sitting on its own
on GitHub, because it is.

## Layout

```text
HAT.md                   # manifest: toolkits, skills, api scope
SOUL.md                  # identity prose the agent reads as system context
README.md                # this file
skills/
  <skill-name>/SKILL.md  # one folder per skill, frontmatter + body
evals/
  smoke.jsonl            # one happy-path case per skill (OpenAI Evals shape)
```

## Adding or editing a skill

1. Create `skills/<slug>/SKILL.md` with valid frontmatter
   (`name`, `description ≤200 chars`, `runtime_tier`, optional
   `runtime_requires`).
2. The frontmatter `name` MUST equal the directory name byte-for-
   byte; the harness fails the mount otherwise (the OpenAI
   Responses API rejects the inline-skill zip if they disagree).
3. Cite endpoints by **operationId**, not URL — operationIds
   survive path renames; URLs don't. The full API map lives in
   [`skills/tinyhat-platform-api-reference/SKILL.md`](skills/tinyhat-platform-api-reference/SKILL.md).
4. Add one happy-path entry to [`evals/smoke.jsonl`](evals/smoke.jsonl)
   so the eval-runner covers the new skill from day one.

The platform runs a frontmatter validator on every mount; a skill
whose frontmatter doesn't parse, whose `name` mismatches the
directory, or whose `description` exceeds 200 chars will fail to
mount and the agent will run without it.

## Lifecycle: in-repo skills → per-skill repos

Individual skills here can graduate to their own GitHub repos
once they're stable and useful for other agents. The flow:

1. Skill ships in this repo at `skills/<name>/SKILL.md`.
2. The platform's `extract-skill` action publishes the skill's
   contents to a new repo and rewrites this hat's `HAT.md`
   `skills:` entry to point at the per-skill repo handle
   (`<org>/<name>`).
3. Other agents can then mount the same skill via
   `add-skill-from-repo`.

The bot-manager's own hat (`HAT.md` + `SOUL.md` in this
directory) typically stays inline — it's the platform-side hat,
unique to this repo. User-owned hats live in their own repos
under their account's GitHub org.

## Reading order for new contributors

1. [`HAT.md`](HAT.md) — what the hat declares.
2. [`SOUL.md`](SOUL.md) — what the agent thinks it is.
3. [`skills/tinyhat-platform-api-reference/SKILL.md`](skills/tinyhat-platform-api-reference/SKILL.md)
   — every API endpoint the bot-manager can reach, by
   operationId.
4. The per-operation skill folders (`skills/<name>/SKILL.md`) —
   workflow patterns the agent follows when the user asks for
   one of those operations.
5. [`evals/smoke.jsonl`](evals/smoke.jsonl) — the test cases the
   eval-runner uses.
