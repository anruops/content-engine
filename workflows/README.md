# Workflows

`content-engine.json` is the full n8n workflow, exported for import into your own n8n instance. It's a single canvas holding three trigger-driven subsystems:

- **Ingest + caption** — Schedule → four RSS feeds → normalize / blocklist / pillar-filter / dedupe → Gemini writes an on-brand caption → row lands in a Google Sheet queue as `Pending`.
- **Publish** — Schedule → pre-flight token check → pull the card image from Drive → post to the Facebook Page feed → publish to Instagram, with an idempotent status machine and a per-platform partial-failure guard.
- **Error handler** — an Error Trigger that catches any failure in either pipeline, throttles repeats, and alerts to Telegram, Slack, and a Google Sheet paper trail.

The two schedules run independently; the Google Sheet queue is the buffer between them, with a human approval step (set a row to `Approved`) in the middle.

## Before you run it

**No tokens or API keys are in this file.** You supply your own two ways:

1. **Re-select credentials.** Every credential-bound node has its ID replaced with a `YOUR_*_CREDENTIAL_ID` placeholder. After importing, open each and pick your own credential (Google service account, Facebook Graph, Gemini, Telegram, Slack).
2. **Replace the ID placeholders** with your own:

| Placeholder | What to put |
|---|---|
| `YOUR_PAGE_ID` | your Facebook Page ID |
| `YOUR_IG_BUSINESS_ACCOUNT_ID` | your Instagram Business account ID (also resolved at runtime by the `Get IG user ID` node) |
| `YOUR_CONTENT_QUEUE_SHEET_ID` | the Google Sheet that holds the content queue |
| `YOUR_ERROR_LOG_SHEET_ID` | the Google Sheet the error handler logs to |
| `YOUR_GRAPHICS_DRIVE_FOLDER_ID` | the Drive folder holding your card images |
| `YOUR_TELEGRAM_CHAT_ID` | the Telegram chat that receives error alerts |
| `#your-error-channel` | the Slack channel that receives error alerts |

**Required Meta token scopes:** `pages_show_list`, `pages_read_engagement`, `pages_manage_posts`, `instagram_basic`, `instagram_content_publish`. Use a never-expiring **Page** token (not a User token — the unpublished-photo upload must act as the Page).

The LLM prompt is written for one specific brand voice; swap it for your own before use.

## Google Sheets setup

The workflow uses two Google Sheets. Create each with a header row (row 1) whose column names match exactly, then put each sheet's ID in the matching placeholder above.

**Content Queue** (`YOUR_CONTENT_QUEUE_SHEET_ID`) — one row per post, moving `Pending` → `Approved` (your manual step) → `Posted`. Header row:

```
title  link  source  pub_date  source_text  pillar  graphic_caption  draft_caption  status  graphic_key  post_order  posted_at  permalink  image_url  ig_posted_at  fb_photo_id  error_note
```

**Error Log** (`YOUR_ERROR_LOG_SHEET_ID`) — one row per unique failure, written by the error handler. Header row:

```
Fingerprint  Timestamp  Workflow  FailedNode  Error  Mode  ExecutionURL  Count
```

> `ig_media_id` is intentionally left out of the Content Queue. If you keep the `Mark Posted` node's default column mapping, delete its `ig_media_id` line too, otherwise the Google Sheets node will re-add the column on the first successful run.

