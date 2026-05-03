---
name: change-model
description: Change which LLM a tinyhat agent runs against (e.g. swap gpt-5 for claude-sonnet-4). Reads available models, confirms, applies. Pending agents.set_model; reply 'not yet shipped' until then.
runtime_tier: 0
---

# change-model

Each agent runs against one upstream LLM. Owners can swap the
model — to chase a price/quality curve, A/B-test voices, or move
off a deprecated model.

## Conversation pattern

1. **Confirm the agent and the target model.** Validate the model
   id against `agents.list_models` (cited in the planned list of
   `tinyhat-platform-api-reference`; until it ships, take the id
   the user gives verbatim). If the user said something vague
   ("the new Claude one"), name the specific id you're going to
   set and wait for yes.
2. **Surface the consequences in one sentence.** "Switching <agent>
   to <model> takes effect on the next message; the harness might
   need a few seconds to load the new client." Most callers just
   want the change; this sentence is for the maintainer who's
   about to demo the bot in five minutes.
3. **Apply.** Call `agents.set_model` with the agent id and the
   new model id.
4. **Report back** with the new model id and a one-line "send
   a message to confirm" instruction.

## When the endpoint isn't shipped yet

`agents.set_model` is planned. Until it ships, you can READ the
current model from `agents.list` (each row exposes `model_provider`
and `model_name`) but cannot change it. Say:

> Current model is <model>. Switching to a different one isn't wired up yet — the endpoint is on the way.

## Do not

- Do not validate model ids against your training data. The list
  the platform supports moves faster than your training; defer
  to whatever the planned `agents.list_models` returns.
- Do not change the model as a side effect of another change.
  Even when "use the cheap one" is implied (e.g. the user is
  asking to lower cost), confirm explicitly before flipping the
  field.
