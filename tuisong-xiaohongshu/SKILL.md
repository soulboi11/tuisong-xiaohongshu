---
name: tuisong-xiaohongshu
description: Monitor one Xiaohongshu creator for new notes that match user-supplied title, text, or video keywords, create source-grounded summaries, and email each match. Use for incremental new-note alerts; do not use to archive or bulk-crawl existing notes.
---

# Tuisong Xiaohongshu

Run one incremental polling cycle for a named Xiaohongshu creator. Email only
new notes that match the requested keywords, and summarize only information
available in the note's title, text, captions, or video transcript.

## Required Inputs

Collect or confirm:

- `creator_name`: the creator's displayed name.
- `xiaohongshu_id`: the creator's Xiaohongshu ID when the profile displays
  one; not a numeric note ID. Do not require it for accounts that do not
  display an ID.
- `keywords`: one or more terms. Default match mode is `any`; use `all` only
  when the user explicitly requests it.
- `recipient_email`: where alerts should be sent.
- Email delivery: an available email connector, or a sender address plus SMTP
  host/port/TLS mode and client authorization code.

If keywords are missing, ask for them. Do not infer keywords from the
creator's historical notes. Do not request or use the Xiaohongshu password.

Treat the email authorization code as a secret. Never write it to the skill,
state file, README, command output, or source control. Prefer an environment
variable or secret store; keep a directly supplied code only in process memory.

## Preconditions

1. Confirm the Xiaohongshu MCP tools are available, including
   `check_login_status`, profile retrieval, and `get_feed_detail`.
2. Call `check_login_status` before fetching creator data. If the tools are
   unavailable or the session is logged out, stop and direct the user to set up
   or log in to `xiaohongshu-mcp`. Do not substitute general web scraping.
3. Confirm an email-sending path before changing polling state. If SMTP is
   used, verify the server, port, TLS mode, sender address, and authorization
   code without echoing the code.

## Resolve the Account

Resolve the creator through the authenticated Xiaohongshu MCP profile or search
tools, then choose the account matching rule from the resolved profile:

- If the profile displays a Xiaohongshu ID, require both the displayed creator
  name and that ID to match exactly.
- If the profile does not display a Xiaohongshu ID, require the displayed
  creator name to match exactly and do not invent or infer an ID.
- Never use a numeric note ID as the creator's Xiaohongshu ID.
- If several accounts share the same displayed name and the profile exposes no
  ID that can distinguish them, do not guess.
- If the MCP cannot resolve the requested account, ask for an official profile
  URL or one recent note URL from that account.
- Fetch only the creator's current visible note list. Do not run a historical
  keyword search or page through old posts.

## Incremental State

Use a user-local JSON state file, by default:

```text
${CODEX_HOME:-~/.codex}/state/tuisong-xiaohongshu/state.json
```

Store one subscription per exact combination of normalized creator name,
Xiaohongshu ID when available, account match mode, recipient email, keyword
set, and match mode. Use `name_and_id` when the profile displays an ID and
`name_only` when it does not. A subscription records:

```json
{
  "creator_name": "example",
  "xiaohongshu_id": "example_id",
  "account_match_mode": "name_and_id",
  "recipient_email": "user@example.com",
  "keywords": ["keyword"],
  "match_mode": "any",
  "baseline_completed_at": "ISO-8601 timestamp",
  "last_checked_at": "ISO-8601 timestamp",
  "seen_feed_ids": [],
  "pending_feed_ids": []
}
```

Store note IDs and delivery status only. Do not store authorization codes or
full note content in the state file.

### First Run

On the first run for a subscription, fetch only the current visible note-list
IDs, save them as `seen_feed_ids`, set `baseline_completed_at`, and send no
email. This prevents existing notes from being treated as updates.

### Later Runs

1. Retry `pending_feed_ids` first. Refetch each pending note, regenerate the
   grounded email, and send it. Remove the ID from `pending_feed_ids` only after
   a successful send; keep it pending if the retry fails.
2. Fetch the creator's current visible note list.
3. Candidates are IDs not already present in `seen_feed_ids` or
   `pending_feed_ids`.
4. Inspect candidates before adding them to `seen_feed_ids`.
5. Add a candidate to `seen_feed_ids` after it is processed, whether or not it
   matches. If its matching email fails, also add it to `pending_feed_ids`.
6. Do not process a successfully emailed note again.

Keep the polling window bounded to the creator's current recent-note listing.
Do not paginate into older content. For continuous monitoring, run the skill
on a schedule that is frequent enough for the creator's posting rate.

## Keyword Matching

Fetch full detail only for candidate notes. Build one source text from:

- title;
- note body;
- visible captions or image text returned by the MCP;
- available video captions or transcript.

Normalize case and insignificant whitespace for Latin text. Match the user's
terms literally unless the user authorizes synonyms. With `any`, at least one
term must occur; with `all`, every term must occur. Record the exact matched
terms.

Do not infer that a video contains a keyword from its cover, music title,
hashtags, comments, or unrelated recommendations.

## Source-Grounded Summary

Summarize only source facts present in the fetched title, text, captions, or
video transcript. Do not add background, product claims, opinions, advice, or
conclusions that the creator did not state.

Use 2-4 short sentences or 3-5 bullets. The email must contain:

- title;
- publish time;
- content summary;
- creator name, and Xiaohongshu ID when available;
- matched keywords;
- note type, when available;
- original note link, when available.

Preserve the source's publish-time value. If only a relative time is available,
keep the relative wording and add the current observation timestamp at the
precision actually available; do not invent missing clock details. If publish
time is not available, write `未获取到`; never invent a date.

For a video:

1. Use the video transcript or captions supplied by the MCP.
2. If media transcription is needed and an approved media tool is available,
   transcribe the video's spoken content.
3. If no transcript or media access is available, do not summarize unseen
   video content. Emit the email with the limitation `视频无可读取字幕或转写`
   and summarize only the accessible title and text.

This video limitation is not the same as a failed note-detail fetch. A failed
detail fetch leaves no usable source and must not be sent.

## Email Delivery

Send one email immediately for each matching new note. If several notes match
in one polling cycle, process and send them individually unless the user asks
for a digest.

Subject:

```text
[小红书更新] {note_title} - {creator_name}
```

Body layout:

```text
创作者：{creator_name}（小红书号：{xiaohongshu_id}）
标题：{note_title}
发布时间：{publish_time}
命中关键词：{matched_keywords}
内容类型：{note_type}
原文链接：{note_url}

内容摘要：
{source_grounded_summary}
```

Omit the Xiaohongshu ID parenthetical from the creator line when the resolved
profile does not display an ID. Omit the link line only when no link was
returned. Add the video limitation line when applicable. Plain text is
sufficient; do not fabricate HTML content.

After the pending queue is processed, do not initiate a new email when:

- there are no new notes;
- no new note matches;
- account resolution is ambiguous;
- the note detail or all usable source content could not be fetched;
- the email delivery path is unavailable.

A pending retry is eligible even when the current run has no new notes; the
conditions above must not suppress that retry.

Mark a matching note as delivered only after the email transport confirms
success. If sending fails, report the failure and keep the note pending for a
retry.

## Final Response

Report counts for fetched candidates, matches, sent emails, and pending or
failed deliveries. On the first run, state that a baseline was created and no
stock notes were emailed.
