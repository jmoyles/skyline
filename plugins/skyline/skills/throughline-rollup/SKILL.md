---
name: throughline-rollup
description: "SkyLine's weekly Throughline rollup: a short plain-text summary of the past 7 days (meetings logged, items closed versus opened, slipping and stalled items, open items by pillar, new decisions) delivered by email. Use when the scheduled weekly rollup fires or when the owner asks for their weekly rollup."
---

# Throughline weekly rollup

Unattended runs have no one to answer questions. Don't ask; proceed.

1. The task passes the Throughline URL (in chat, find the artifact titled "Throughline" with Artifact action "list"). Read `meta/config` for `ownerName`, `timezone`, `pillarWord`, and `pillars`.
2. `read_db` with db_op `list`: `action_items` (limit 1000), `meetings` (limit 500), `decisions` (limit 500). Ignore archived docs.
3. Over the last 7 calendar days in the owner's timezone, compute: meetings logged (count, with titles and dates; skip stubs), decisions recorded (count), action items closed (status "done" with updatedAt in the window), action items newly raised (createdAt in the window), and slipping items (still open with a past dueDate, oldest due first). Separately call out stalled items (timesCarried of 3 or more, not done), even if already listed.
4. Count open items per pillar using the labels in `config.pillars`, plus "Unset". If `pillars` is empty, skip this section.
5. Compose the rollup: a one-line headline (meetings logged; items closed versus opened, net), then short sections for Slipping, Stalled, the pillar breakdown (headed with `pillarWord`), and New decisions worth remembering. Omit empty sections. Under about 25 lines. Plain text only: no markdown, no HTML, capitalized labels on their own lines, hyphen-space for list items. No preamble. Add the Throughline URL as the last line.
6. **Deliver it.** In a scheduled run, call PushNotification exactly once with status "proactive", message wrapped in `<routine_summary>...</routine_summary>`. A final chat message alone does not send email. Then repeat the same text as your final chat message for the run log. In an interactive chat, just show the rollup.
