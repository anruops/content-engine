# Autonomous Multi-Platform Content Engine

A self-hosted [n8n](https://n8n.io) system that ingests industry news, drafts on-brand social captions with an LLM, and publishes to Facebook and Instagram on a schedule. No human in the loop after approval, and real-time alerting the moment anything breaks.

> **Status:** Built and running (Sept 2026), maintained in production. This repository documents the architecture and shares **sanitized** material. It's a portfolio reference and a build I run for my own brand, not a one-click install.

---

## Why it exists

Staying visible on social media as a solo operator is a grind: read the news, decide what's worth posting, write a caption in a consistent voice, and post to each platform separately, every day. I wanted the output of a daily content team without the daily labor, built on tools I control rather than another rented SaaS, and reliable enough to keep running when I ignore it for a week.

## Architecture

```
  RSS sources ─┐
               ▼
   [Workflow A · daily]  normalize → dedupe → pillar-filter → freshness cap
               │         → LLM caption (batched) → write row (status: Pending)
               ▼
        Content Queue (Google Sheet)  ── I approve the good rows ──┐
                                                                    ▼
   [Workflow B · scheduled]  read Approved → order → one at a time:
        mark Posting → pull graphic → post to Facebook feed
        → read back public image URL → create IG container → publish
        → mark Posted
               │
               ▼
     [Error Handler · shared]  any failure → Telegram + Slack + a logged paper trail
```

## How it works

- **Workflow A — ingest and write.** Pulls RSS feeds, normalizes and de-duplicates against the existing queue, filters each item to a content pillar, caps how much any one feed can contribute so the mix stays balanced, then hands a batch to the LLM to write captions in a fixed brand voice. Results land in a Google Sheet as `Pending`. I curate; I do not write.
- **Workflow B — publish.** Reads only approved rows, orders them, and processes one per run so a single fire equals a single post. It walks a status machine, pulls the matching graphic from Drive, posts a real Facebook feed post, then handles Instagram (which cannot take image bytes directly) by reading the public image URL back from Facebook and passing it to Instagram's two-step publish.
- **Error Handler — the safety net.** A shared workflow that catches any failure in either pipeline and pushes it to Telegram and Slack with a logged trail, so a silent break is impossible.

## Engineering decisions worth calling out

- **Idempotent status machine.** Every row moves `Approved → Posting → Posted`, with distinct terminal states for partial success (`Posted-FB`) and total failure (`Failed-FB`). A crashed run can never double-post, and a half-finished post resumes from exactly where it stopped.
- **Partial-failure guard.** Publishing hits two platforms, so "posted to one, failed on the other" is a first-class case: the failing path writes a terminal status plus an error note, then re-raises so the global alert still fires.
- **Rate-limit engineering, not brute force.** The LLM is called in small batches with deliberate delays to stay under free-tier limits, and ingestion runs once daily on a freshness cap rather than draining the source archive.
- **Token lifecycle done properly.** Authenticates with a never-expiring Page token, which requires extending a short-lived token to long-lived *before* deriving the Page token, or it silently expires within the hour.

## A debugging pass from production

After the Meta app moved to Live mode, posting failed on Facebook while Instagram kept publishing: FB returned `(#200) Unpublished posts must be posted to a page as the page itself`. That one-sided failure was the clue. Instagram tolerates a User access token, so IG posting is what stays green when the token is wrong, which isolated the problem to token identity rather than the workflow. The unpublished FB upload has to act as the Page, and the credential was set to a User token; the fix was pointing it at the Page token. The rule it reinforces: when a multi-platform automation fails on one platform only, suspect token identity before touching the flow.

## Stack

n8n (self-hosted, Node.js, behind a Cloudflare tunnel) · Meta Graph API (Facebook + Instagram) · an LLM caption model · Google Sheets + Drive · Telegram + Slack for alerting.

## Repository structure

```
README.md      this file
docs/          full case study + notes
assets/        screenshots of the workflow canvas and a run  (add your images here)
workflows/     sanitized n8n export (JSON)                    (add after sanitizing)
```

## A note on credentials (read before cloning)

No secrets are committed. Tokens, Page/IG IDs, sheet and Drive IDs, and connection credentials live in the n8n credential store and must be **stripped from any exported workflow** before it goes in `workflows/`. If you adapt this, you supply your own.

## What this demonstrates

Multi-service API integration, OAuth and access-token lifecycle management, idempotent and fault-tolerant workflow design with explicit state, practical LLM integration under real rate limits, self-hosted infrastructure, and observability through centralized error handling with real-time alerts.
