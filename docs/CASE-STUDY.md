# Case Study — An Autonomous Multi-Platform Content Engine

*Systems & AI Automation Engineer · Built and running as of 2026-09, maintained in production. No fabricated metrics; results are added only as real data accrues.*

---

## One line

A self-hosted system that ingests industry news, writes on-brand social captions with an LLM, and publishes to Facebook and Instagram on a schedule, with no human in the loop after approval, and error handling that alerts the moment anything breaks.

## The problem

Staying visible on social media as a solo operator is repetitive, high-friction work that quietly doesn't get done: read the news, decide what's worth posting, write a caption in a consistent voice, design a card, post it to each platform separately, every day. I wanted the output of a daily content team without the daily labor, built on tools I control rather than another rented SaaS.

## Constraints I designed around

- **Solo and self-hosted.** Runs on my own n8n instance (Node.js behind a Cloudflare tunnel), not a paid automation platform.
- **Free-tier APIs.** The LLM and the social APIs both have rate and quota limits I had to engineer around, not spend past.
- **Honest by rule.** Every caption is generated from real source material.
- **It has to survive being ignored.** If I don't look at it for a week, it either keeps working or tells me exactly why it stopped.

## Engineering decisions worth calling out

- **Idempotent status machine.** Every row moves `Approved → Posting → Posted`, with distinct terminal states for partial success (`Posted-FB`) and total failure (`Failed-FB`). A crashed run can never double-post; a half-finished post resumes from exactly where it stopped.
- **Partial-failure guard.** Publishing hits two platforms, so "posted to one, failed on the other" is a first-class case: the failing path writes a terminal status plus an error note, then re-raises so the global alert still fires.
- **Rate-limit engineering, not brute force.** The LLM is called in small batches with deliberate delays to stay under free-tier limits, and ingestion runs once daily on a freshness cap rather than draining the whole source archive (an early version without that cap ballooned the queue past a thousand rows and burned quota).
- **Token lifecycle done properly.** Authenticates with a never-expiring Page token, which requires extending a short-lived token to long-lived *before* deriving the Page token, or it silently expires within the hour.
- **Reliability over cleverness.** A code node for the fiddly logic, an explicit status column for idempotency, a pre-flight token check so a bad credential stops the run cleanly instead of half-posting.

## Outcome

Live and fully autonomous: Workflow A curates overnight, I approve rows, and Workflow B publishes to Facebook and Instagram on a schedule on its own, with every failure path wired to alert in real time.

*Results to add as they accrue: posts published to date, reach/engagement trend, hours saved per week.*

## A production debugging pass (2026-09-30)

Keeping a live system running is part of the build. After the app was moved to Live mode, posting failed on Facebook while Instagram kept publishing: FB returned `(#200) Unpublished posts must be posted to a page as the page itself`. The one-sided failure was the clue. Instagram tolerates a User access token, so IG posting is exactly what stays green when the token is wrong, which isolated the problem to token identity rather than the workflow. The unpublished FB photo upload has to act as the Page, and the credential was set to a User token. The fix was a single setting: point the credential at the Page token. The engine returned to unattended posting on both platforms.

The rule it reinforces: when a multi-platform automation fails on one platform only, suspect token identity before touching the flow.

## What this demonstrates

- Multi-service API integration (Meta Graph API for two platforms, an LLM API, Google Sheets and Drive)
- OAuth and access-token lifecycle management, including the failure modes
- Idempotent, fault-tolerant workflow design with explicit state and partial-failure handling
- Practical LLM integration under real rate limits
- Self-hosted infrastructure (n8n on Node.js behind Cloudflare) rather than rented SaaS
- Observability: centralized error handling with real-time alerting and a logged trail

## Stack

n8n (self-hosted) · Meta Graph API (Facebook + Instagram) · an LLM caption model · Google Sheets + Drive · Telegram + Slack (alerting) · Cloudflare tunnel.
