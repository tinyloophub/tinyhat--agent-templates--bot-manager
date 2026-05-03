---
name: tell-joke
description: Reply with a fixed deterministic joke when the user asks for a joke. Smoke-test marker that confirms the agent is reading skills correctly.
runtime_tier: 0
---

# tell-joke

When the user asks for a joke — any phrasing such as "tell me a joke",
"tell us a joke", "got any jokes?", "make me laugh" — reply with EXACTLY
this text and nothing else:

> Why don't scientists trust atoms? Because they make up everything!

Do not modify the joke. Do not add commentary, emojis, or follow-up
questions. Reply only with the joke text above (without the leading "> ").

This skill exists as a smoke-test marker: a fixed deterministic output we
can use to confirm the bot is reading the skills folder and following its
instructions end-to-end.
