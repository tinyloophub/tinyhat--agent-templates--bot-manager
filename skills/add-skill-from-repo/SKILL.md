---
name: add-skill-from-repo
description: Mount an external skill onto a tinyhat agent by cloning a per-skill repo (handle like `tinyloophub-tinyhat-skills/email-sender`). Pending skills.add_from_handle; reply 'not yet shipped' until then.
runtime_tier: 0
---

# add-skill-from-repo

The owner wants to add a skill from one of the published per-skill
repos. The handle shape is `<github-org>/<repo-name>` (e.g.
`tinyloophub-tinyhat-skills/email-sender`). The platform clones
the repo, vendor-pins it on disk, and adds an entry to the hat's
skills frontmatter.

## Conversation pattern

1. **Take the handle** from the user. Validate the shape (looks
   like `org/repo`). If it doesn't, ask once more.
2. **Confirm the target hat.** Only the hat's owner may add
   skills to it.
3. **One-line confirm** the addition, naming both. Wait for yes.
4. **Call `skills.add_from_handle`.** Body: `{"handle": "...",
   "agent_id": <id>}`. The platform handles the clone, the
   validation pass, and the vendored copy.
5. **Report success or failure** with the platform's reason
   verbatim (e.g. "skill rejected: SKILL.md missing
   `runtime_tier`"). Do not invent reasons; the platform's
   message is the user's truth.

## When the endpoint isn't shipped yet

`skills.add_from_handle` is planned. Until it ships, say:

> Adding skills from external repos isn't wired up yet. Once the platform's skill-handle endpoint lands I'll be able to mount <handle> onto your agent.

Suggest the user wait for that issue rather than trying to clone
the repo themselves; pre-vendor manual edits are the kind of state
the platform's bookkeeping won't see.

## Do not

- Do not fetch the repo yourself via `gated_api_call`. The platform
  has its own clone-and-validate path; running our own would skip
  the validation pass and could mount a malformed skill.
- Do not assume skills are immediately mounted. Existing agent
  runs continue with the old skill set; the next invocation reads
  the new manifest.
