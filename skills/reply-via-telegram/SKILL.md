---
name: reply-via-telegram
description: How to compose a reply for Telegram and send it via the `reply_via_telegram` function tool. Read this before you reply.
runtime_tier: 0
---

# reply-via-telegram

This skill is **documentation**, not code. The actual delivery
primitive is the `reply_via_telegram(text)` **function tool** in
your tool catalog (provided by the `bot-manager-20260427` toolkit).
This skill body tells you *how to compose* the message you pass
into that tool.

## How to send a reply

Call the function tool exactly once per user message:

```
reply_via_telegram(text="<your message here>")
```

That's it. No curl, no `source`, no env vars. The platform handles
the bot token, the chat id, and audit logging. You don't see any
of it.

## Composition rules

- **Plain text by default.** Telegram renders plain text well; only
  reach for formatting when the message benefits from it.
- **Length.** Keep replies short (one or two sentences for a chat
  exchange; up to ~5 short paragraphs for an explanation). The
  tool truncates to 3997 chars + "…" if you go over Telegram's
  4096-char hard cap, but **do not rely on truncation** —
  truncating mid-thought looks broken.
- **No secrets.** Never include tokens or anything that looks like
  one (`Bearer …`, `eyJ…`, `$TINYLOOP_…`). The user must not see
  these.
- **Match the user's register.** If they're casual, be casual. If
  they're using full sentences, you do too.

## Formatting (when you do want it)

The tool sends plain text by default. If you want Telegram-styled
text, the function tool currently accepts only `text` — formatting
options (`parse_mode`, `reply_to_message_id`, attachments) will
land as additional function-tool arguments in a future toolkit
version (`bot-manager-20260601`+). Until then, stick to plain text.

When that day comes, the convention will be:

- `parse_mode="MarkdownV2"` (or `"Markdown"`, or `"HTML"`) for
  styled text. Each parse mode has strict escaping rules; a single
  unescaped character rejects the whole message. Default is plain.
- `reply_to_message_id=<int>` to thread the reply under a specific
  inbound message id.
- Attachments (photos, documents) will be a separate function tool
  (`reply_via_telegram_with_image(text, image_url)` or similar);
  not in scope today.

## What to put in your final assistant message

The harness has a watchdog. If your call to `reply_via_telegram`
errors (the function tool raises, or it returns
`{"delivered": false}`), the harness reads your **final assistant
message** and delivers it to the user directly via Telegram. So
your final assistant text must be:

- **The user-facing reply itself.** Write it as the message you'd
  want the user to read. Not "Sent the joke." — write the joke.
- **Match the text you passed to the tool.** Same content, same
  voice. The user sees one or the other depending on which path
  delivered; neither path should look different.

When the function tool succeeds, your final assistant text is just
an audit log entry. When it doesn't, it's what the user sees.
Either way the text is the same — write it well, every time.

## Why this is a SKILL.md and not "just a tool"

A function tool's docstring tells the model *what the tool does*
and *what arguments it takes*. This SKILL.md tells the model *how
to use it well in this product's voice* — the editorial part
(length, register, no-secret rule, formatting guidance, watchdog
contract) that doesn't fit cleanly into a tool signature.

Splitting them this way means:

- The function-tool signature stays tight and stable.
- This skill body can grow as we learn what users like / don't
  like, without re-versioning the toolkit.
- A future hat that uses the same function tool can mount its own
  variant of this skill with a different voice / set of rules.
