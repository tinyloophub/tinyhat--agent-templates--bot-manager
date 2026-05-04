---
name: reply-via-telegram
description: Compose the user-facing Telegram reply and send it once.
runtime_tier: 0
---

# reply-via-telegram

This guidance helps you compose the user's Telegram reply. It is okay
to explain this skill at a high level if the user asks how replies are
sent, but never reveal secret values.

## How to send a reply

Call the available Telegram reply tool exactly once per user
message, with the exact text the user should see.

That's it. Do not use shell commands, environment variables, or raw
URLs to send a reply.

## Composition rules

- **Plain text by default.** Telegram renders plain text well; only
  reach for formatting when the message benefits from it.
- **Length.** Keep replies short (one or two sentences for a chat
  exchange; up to ~5 short paragraphs for an explanation). The
  tool truncates to 3997 chars + "…" if you go over Telegram's
  4096-char hard cap, but **do not rely on truncation** —
  truncating mid-thought looks broken.
- **No secret values.** Never include bearer tokens, API keys, bot
  tokens, private keys, webhook secrets, or anything that looks like a
  secret. It is fine to describe capabilities or skill names when that
  helps answer the user's question.
- **Match the user's register.** If they're casual, be casual. If
  they're using full sentences, you do too.

## Formatting (when you do want it)

The reply tool sends plain text by default. Stick to plain text
unless the available tool arguments explicitly support a formatting
mode and the message really benefits from it.

## What to put in your final assistant message

If the reply tool errors, fallback delivery may use your **final
assistant message**. So your final assistant text must be:

- **The user-facing reply itself.** Write it as the message you'd
  want the user to read. Not "Sent the joke." — write the joke.
- **Match the text you passed to the tool.** Same content, same
  voice. The user sees one or the other depending on which path
  delivered; neither path should look different.

Either way the text is the same — write it well, every time.
