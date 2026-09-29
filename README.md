# SkyLine

SkyLine is a meeting, decision, and commitment tracking suite for Claude.

- **Throughline** keeps a live record of your meetings, decisions, and action items, pulled automatically from your Microsoft 365 calendar and email (and Fellow, if you use it). It lives on a private page only you can see.
- **Ridgeline** turns that record into a triaged daily digest: what's overdue, what's due this week, what's stalled, and what still needs a date, plus trip prep when you're traveling. It arrives by email each morning.
- A **weekly rollup** lands every Monday.

## Requirements

- A Claude plan with Cowork or Claude Code (setup has to run there, because that's where scheduled tasks are created)
- A Microsoft 365 work account
- Optional: a Fellow account

## Install

1. In Claude, open **Customize**, then **Plugins**.
2. Select **Add marketplace** and enter this repository's URL.
3. Find **SkyLine** and select **Install**. Sign in to Microsoft 365 when prompted.
4. If you use Fellow, connect the Fellow connector in **Customize**, **Connectors**.
5. Start a Cowork session and run `/skyline:setup`. Answer one round of questions (or say "defaults"). Setup publishes your pages, schedules everything, and runs a first test pull.

That's it. Bookmark the Throughline link setup gives you.

## Everyday use

- Paste or attach a meeting transcript in any chat to log it (Teams transcripts included).
- Ask things like "what's overdue", "what did we decide about X", or "prep me for my meeting with Pat".
- `/skyline:ingest-now` pulls new items immediately instead of waiting for the next run.
- `/skyline:ridgeline` refreshes your digest on demand.
- `/skyline:status` checks that everything is running.

## Updates

When a new version is released, open **Customize**, **Plugins**, and select **Update** on the SkyLine marketplace. Most improvements take effect immediately. If the release notes say so, run `/skyline:upgrade` to refresh your pages and tasks; your data is never touched.

## Removing it

Run `/skyline:uninstall` to stop the scheduled tasks, then remove the plugin in **Customize**, **Plugins**. Your Throughline and Ridgeline pages stay until you delete them.

## Privacy

Everything SkyLine records lives in your own Claude account, on pages private to you unless you choose to share them. SkyLine reads your calendar, Inbox, Sent Items, and Drafts. It never opens email attachments and never sends email or messages on your behalf.
