---
name: slack
description: How to send Slack messages from this agent. Use whenever the user asks you to "send to Slack", "post to Slack", "DM the channel", "ping me on Slack", reply to a Slack thread, share a file in Slack, or otherwise communicate via Slack from this session.
---

# Working with Slack

Slack messaging from this agent goes through the `mcp__humr-outbound__send_channel_message` tool. The runtime knows which Slack chat is currently connected to this agent instance, so in the common case you do **not** need to look up channel IDs or workspaces — just send.

## The default path: just send

For 95% of requests ("tell the channel X", "let me know in Slack when done", "reply to that"), call `send_channel_message` with `channel: "slack"` and `text`. Omit `chatId` — the tool routes to the last-active chat for this agent. Do NOT call `describe_channel` first; it's an unnecessary round trip.

```
mcp__humr-outbound__send_channel_message({
  channel: "slack",
  text: "Build finished — all 142 tests passed."
})
```

## When to call `describe_channel` first

Only when the user is explicit that they want a *different* chat than the active one — e.g. "post this in #releases too", "send it to the eng-leads DM, not here". Then:

1. Call `mcp__humr-outbound__describe_channel({ channel: "slack" })` → returns `{ chats: [{ id, title }] }`.
2. Match the user's wording against `title` to pick the right `id`.
3. Pass it as `chatId` in `send_channel_message`.

If the lookup is ambiguous (multiple plausible matches, or none), ask the user which one rather than guessing.

## Attachments

To attach a file, set `attachment.path` — absolute (e.g. `/home/agent/work/report.md`) or workspace-relative (`report.md`). 10 MiB cap, one file per message. Optional `attachment.filename`, `attachment.title`, `attachment.mimeType` override defaults.

```
mcp__humr-outbound__send_channel_message({
  channel: "slack",
  text: "Latest perf numbers attached.",
  attachment: { path: "bench/results.csv", title: "Bench results" }
})
```

If the user asks you to "share the file" or "send the diff", prefer attaching the file over pasting its contents inline — Slack handles long content poorly inline.

## Message style

- Slack messages are read on phones and in busy channels. Lead with the headline; keep it tight.
- Use Slack's `mrkdwn` (single `*bold*`, `_italic_`, `` `code` ``, ```` ``` ```` for blocks). Do NOT use GitHub-flavored markdown headings (`#`, `##`) — they render as literal `#` characters.
- For multi-line status updates, prefer a one-line summary then bullets. Don't paste full logs unless asked; attach them instead.
- No need to sign messages — the channel already attributes them to this agent.

## Common pitfalls

- **Don't call `describe_channel` reflexively.** The active-chat default is the right answer almost every time, and the lookup costs a tool call and clutters context.
- **Don't ask the user to confirm before sending** when they've already asked for the message to go out — sending is the action they requested. Confirm only if the content is irreversible/sensitive (publishing a release announcement, paging on-call, etc.) or if you had to make a non-trivial judgment call about what to say.
- **Telegram uses the same tool** with `channel: "telegram"`. If the user says "telegram" instead of "slack", switch the channel value; everything else works the same.
