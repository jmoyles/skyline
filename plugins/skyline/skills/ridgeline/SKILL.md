---
name: ridgeline
description: "SkyLine's Ridgeline daily digest: a triaged morning briefing built from the owner's Throughline tracker plus live calendar and email cross-checks, published to a permanent Ridgeline page and delivered as a short email. Use when a scheduled Ridgeline run fires, or when the owner asks to run, refresh, or see their Ridgeline digest."
---

# Ridgeline

Ridgeline is an EA-style morning briefing that triages and prioritizes. It is not a raw dump of the tracker. Throughline is the system of record: don't re-derive its ingestion logic, don't re-log meetings, and don't second-guess its data except as noted below. You do not need to load the throughline skill for a Ridgeline run.

Unattended runs have no one to answer questions. Make the reasonable call and proceed.

## Inputs

The scheduled task passes the Throughline URL. If it's missing (a chat request), find the artifact titled "Throughline" with Artifact action "list". Read `meta/config` from it for `ownerName`, `company`, `timezone`, and `ridgelineUrl`. "Today" is the date this run fires in the owner's timezone.

## Data gathering

1. Read every action item: `read_db`, collection `action_items`, db_op `list`, query.limit 1000.
2. Check the owner's calendar (`outlook_calendar_search`, query `*`) for the next 5 days for a travel-pattern event starting within 3 days: titles containing Travel, Trip, Office Visit, Out of Office, or a multi-day all-day hold. If one exists, that is today's Trip. If none, there's no Trip section.
3. If there is a Trip: search the Inbox (last ~7 days) for flight, hotel, and ground-transportation confirmations covering those dates. Read full bodies via `read_resource` before concluding anything is missing; forwarded confirmations often carry both legs. Also check whether the calendar has real meeting invites with the people the trip is for, not just a personal hold. Never fabricate a confirmation or meeting. Flag plainly, not alarmingly, when a leg, hotel, ground transport, or meeting invite isn't found. Never open attachments.

## Triage

For every action item where status is not "done" and not archived:
- OVERDUE: dueDate before today.
- DUE SOON: dueDate today through 7 days out.
- STALLED: timesCarried of 3 or more, regardless of due date.
- NO DUE DATE: dueDate null or empty.

An item can land in more than one bucket.

## Sections

Omit any section with nothing to report. Never render an empty shell.

- **TODAY**: every OVERDUE item, plus anything from the Trip cross-check that is genuinely time-critical today. EA voice: one short line of real context, one short line of what to do. No hedging, no invented consequences, no filler.
- **TRIP PREP** (only with a Trip): itinerary style, each leg, hotel, and ground transport as its own line with a status, either confirmed (with the real confirmation detail) or "not found, flagged". Add a line if the calendar shows only a hold rather than real invites, and how many nights to pack for.
- **THIS WEEK**: DUE SOON items not already in Today, soonest first: description, owner, direction (I owe / owed to me), due date, days out.
- **STALLED**: STALLED items not already in Today. For each, synthesize from its `notes` into four short lines: What (the actual ask), Why stuck (concretely), Owner (whose court it's in now), Restart (what unsticks it). If two entries clearly describe the same real item, say so once and treat them as one card.
- **NEEDS A DATE FROM YOU**: remaining NO DUE DATE items, grouped High / Medium / Low by `priority` (missing priority counts as Medium). One line per item with days open (today minus raisedDate, falling back to createdAt). Always show the full list, never a count or sample. Close with one short line noting it shrinks only when the owner dates or kills items.
- If nothing qualifies anywhere and there's no trip, the page says so in one centered all-clear line.

## Exception write-back (rare)

Ingestion keeps the tracker current; don't duplicate its work. Write to `action_items` only when the Trip cross-check finds something clearly missing and clearly actionable that ingestion had no way to catch, such as unbooked ground transport for an imminent trip. Check for an existing match first and update it rather than duplicate. Give new items a real dueDate, `category: "logistics"` where relevant, and notes on what was found. Pin writes with `if_version`; if a pinned write is rejected, re-read and redo it once. Silence is the normal case.

## Build the page

1. Call Artifact action "read" on `ridgelineUrl` first. It's required before publishing to an artifact this session hasn't read.
2. Start from `template.html` in this skill's folder (locate it with `find / -path "*ridgeline/template.html" 2>/dev/null` if needed). Keep its CSS, fonts, and component classes exactly. Replace only content: the masthead date, `[OWNER NAME]` and `[COMPANY]` from config, the four stat numbers (Today count, Trip as a days-out number or a dash, due this week, no date), the section bodies, and the footer date. Leave omitted sections out of the markup entirely. For an all-clear day, replace the section stack with `<div class="allclear">...</div>` and keep the masthead and stats.
3. Write the result to a file under /mnt/user-data/outputs/ and publish with Artifact, passing `url` set to `ridgelineUrl`, title "Ridgeline", and favicon ⛰️. Never omit `url`; that would fork a new page and break the permanent link.

## Deliver the email

A final chat message alone does not send an email. The PushNotification tool is the only reliable delivery path in scheduled runs.

1. Call PushNotification exactly once with status "proactive". Wrap the message in `<routine_summary>...</routine_summary>`. Content must be plain text: no HTML, no markdown. The email template drops text in verbatim, so markup shows as literal punctuation. Use short capitalized labels on their own lines, blank lines between sections, and a hyphen-space for list items.
   - First line: the headline counts (today, trip, due this week, needs a date).
   - Today's items in full, two or three short lines each.
   - One line on the Trip if there is one, naming the most important gap.
   - Close with: "Full digest (this week, stalled items, and the no-date list): " followed by `ridgelineUrl`.
   - On an all-clear day: one short line plus the link.
   - EA voice, no preamble.
2. Then write the same content as your final chat message for the run log.

In an interactive chat request, skip PushNotification (the owner is right there) and just summarize with the link.
