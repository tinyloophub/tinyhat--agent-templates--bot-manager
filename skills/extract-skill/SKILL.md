---
name: extract-skill
description: Take an in-tree tinyhat skill and publish it as its own per-skill GitHub repo so other hats can mount it via add-skill-from-repo. Pending skills.extract; reply 'not yet shipped' until then.
runtime_tier: 0
---

# extract-skill

A skill that lives inline in an agent's repo (under
`skills/<name>/SKILL.md`) can be promoted to its own per-skill
GitHub repo so other agents and other operators can pick it up.
This skill walks that workflow.

The platform-side flow is roughly:
1. Read the in-tree skill folder.
2. Push its contents to a new repo under the
   `tinyloophub-tinyhat-skills/` org (or a maintainer-supplied
   org/name).
3. Replace the source hat's `skills:` frontmatter entry with the
   `<org>/<repo>` handle so it now mounts via
   `add-skill-from-repo` like any other external skill.

## Conversation pattern

1. **Take the source.** Need the source hat name AND the in-tree
   skill folder name. If the user gave one and not the other, ask
   for the missing piece.
2. **Take the destination.** Default org is
   `tinyloophub-tinyhat-skills`; default repo name is the skill
   folder name. Confirm both back.
3. **Sanity-check by reading the SKILL.md** (use whatever the
   shipped equivalent of `skills.list_for_agent` returns). If the
   description is empty or it has unresolved TODOs in the body,
   tell the user to fix those first.
4. **Echo the plan once.** "I'll publish `customize-soul` from
   the bot-manager hat to
   `tinyloophub-tinyhat-skills/customize-soul` and update the
   bot-manager's manifest to point at the new handle." Wait for
   yes.
5. **Call `skills.extract`.** The endpoint owns the git plumbing
   and the manifest swap; report back its returned repo URL.

## When the endpoint isn't shipped yet

`skills.extract` is planned. Until it ships, say:

> Extracting a skill into its own repo isn't wired up yet. The endpoint that does the publish-and-rewire is on the way; until then the in-tree path is the only home this skill has.

Don't try to do the publish manually with a raw GitHub API call;
that would skip the manifest rewrite and leave the source hat
mounting from the in-tree path while a stale copy lives in the
new repo.

## Do not

- Do not extract a skill that depends on private internals (e.g.
  hardcoded paths to our backend). The shared per-skill repo
  contract is "any tinyhat hat can mount this"; an extract that
  silently broke that invariant is worse than no extract.
- Do not rename the skill during extraction. The handle changes
  enough as it is; double-rename and you lose the audit trail.
