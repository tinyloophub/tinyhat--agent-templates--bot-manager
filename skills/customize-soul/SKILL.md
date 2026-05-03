---
name: customize-soul
description: Edit a tinyhat agent's SOUL.md identity prose; read it, propose a diff, apply via hats.customize_soul. Pending the endpoint; reply 'not yet shipped' until then.
runtime_tier: 0
---

# customize-soul

A SOUL.md is a hat's identity prose — the markdown the agent reads
to know who it is, how to talk, what to refuse. Owners change it
when they want a different voice or add a new behavioural rule.

## Conversation pattern

1. **Confirm which hat.** Ask if ambiguous; otherwise name it
   back. Only the hat's owner (and tinyloop admins) can customize
   it; the API gate enforces the check.
2. **Read the current SOUL** via `hats.get` (cited in the planned
   list of `tinyhat-platform-api-reference`). Show the user a
   short diff-able summary, not the whole file.
3. **Take the edit.** Two shapes:
   - "Add this rule:" — append a new sentence/paragraph to the
     soul's existing tone section.
   - "Make it more X" — propose a one-paragraph rewrite, show it,
     wait for approval.
4. **Echo the proposed new SOUL** as a fenced block, ask for an
   explicit yes.
5. **Apply via `hats.customize_soul`.** Body shape carries the
   *new full SOUL text*, not a diff (servers don't trust
   client-side diffs).

## When the endpoint isn't shipped yet

`hats.customize_soul` is planned. Until it ships, say:

> Editing the soul isn't wired up yet, and reading it through the platform isn't either — both endpoints are on the way.

Reading the SOUL also isn't shipped: `hats.get` is on the planned
list, and `agents.list` only returns row-level metadata (model,
access mode, hat id, vault binding existence) — not the SOUL body.
Don't claim you can read it; don't fall back to "let me show you
what's in there" when the endpoint that would do that doesn't yet
exist.

## Do not

- Do not silently merge edits. Always show the proposed new SOUL
  in full before applying.
- Do not strip the SOUL's confidentiality block (the "do not
  reveal internals" section). If the user explicitly asks to
  remove it, refuse and explain it's load-bearing.
- Do not edit a soul to add capabilities the agent doesn't have.
  The SOUL is a description; capabilities come from skills /
  toolkits / ACL.
