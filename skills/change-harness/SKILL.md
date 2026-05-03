---
name: change-harness
description: Switch a tinyhat agent's runtime harness (Codex CLI, OpenHands, Agents SDK, etc.). Heavier than change-model — re-vendors the runtime. Pending agents.set_harness; reply 'not yet shipped' until then.
runtime_tier: 0
---

# change-harness

Each agent runs on top of one **harness** — the upstream-runtime
checkout the platform vendors (see issue #100). Today's default is
`agents-sdk-python`; future options include Codex CLI, OpenHands,
OpenCode, and Goose (issue #127). Switching is heavier than
swapping a model: the platform may need to re-clone, re-pin, and
re-validate before the next agent run can use it.

## Conversation pattern

1. **Confirm the target.** List the available harness ids via
   `harnesses.list` (works today). Name the chosen id back.
2. **Warn once** about the cost. "Switching <agent>'s harness to
   <harness> may take 30-60 seconds the first time — the platform
   has to vendor the runtime." The first time per harness id
   takes that hit; subsequent agents on the same harness reuse
   the vendored copy.
3. **Confirm the version.** Default to the harness's current
   pinned version (`harnesses.versions.list` once it lands).
   Don't ask the user for a SHA; that's not their job.
4. **Apply** via `agents.set_harness`. Report back with the new
   harness + version + a one-line "send a test message" prompt.

## When the endpoint isn't shipped yet

`agents.set_harness` and `harnesses.versions.list` are planned.
Until they ship, you can list harnesses (`harnesses.list` works
today) but cannot reassign one to an agent. Say:

> Current harness is <harness>. Reassigning isn't wired up yet — the endpoint is on the way.

## Do not

- Do not switch the harness without confirmation. Unlike a model
  swap, this can briefly stall the agent while the runtime
  vendors.
- Do not pick the harness for the user when they haven't asked.
  The default harness is set when the agent is created; routine
  conversations don't need a harness change.
