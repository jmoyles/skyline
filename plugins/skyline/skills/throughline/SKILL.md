---
name: throughline
description: "SkyLine's Throughline tracker: the owner's live record of meetings, decisions, and action items, kept in a hosted Throughline artifact. Use when logging a meeting transcript or notes, running the external ingestion pipeline (calendar, email, Fellow), answering questions about meeting history, open items, overdue or stalled commitments, or prepping for a meeting. Also used by SkyLine's scheduled ingestion tasks. If no Throughline artifact exists yet, point the person to /skyline:setup."
---

# Throughline

Throughline is a record-keeping tool for one person (the "owner"): memory and follow-through for their meetings and commitments. It is not an agent that acts on their behalf. It cannot schedule meetings, send email, or post messages. The living record is a hosted artifact with a shared database, not the chat.

## Step zero: find the artifact and read the config

1. **Artifact URL.** Scheduled tasks pass it in their prompt; use that. In chat, call the Artifact tool with action "list" (scope "mine") and pick the artifact titled "Throughline". If there is none, stop and tell the person to run `/skyline:setup`. If there are several, use the most recently updated one and mention the others.
2. **Config.** Read `meta/config` with `read_db` (collection `meta`, doc_id `config`). Every person-specific fact comes from here, never from this file:
   - `ownerName`, `ownerTitle`, `company`, `email`, `timezone` (IANA, e.g. America/New_York)
   - `pillarWord` (what the owner calls their categories, default "Pillars") and `pillars` (array of `{key, label}`, may be empty)
   - `sources`: `{ calendar, email, fellow }` booleans for what ingestion should pull
   - `teamsTranscripts`: `"manual"` (default) or `"graph"` if setup confirmed Graph transcript access works for this tenant
   - `ridgelineUrl`, `throughlineUrl`, `installedVersion`
   - `notes` (optional array of strings): owner-specific facts learned over time, such as known diarization mix-ups or which Fellow account is live. Read them and follow them. When the owner teaches you a durable fact like this, append it here rather than trying to edit the skill.

Throughout this file, "the owner" means `ownerName`. All dates and "today" are in the owner's `timezone`.

## Schema

**`meetings`** (doc id: short slug like `2026-09-15-ops-standup`): `date` (YYYY-MM-DD), `title`, `attendees` (array), `summary` (3 to 6 sentences), `topics` (array of free tags), `pillars` (array of pillar keys from config), `highlights` (array), `riskFlags` (array), `otherActionItems` (array of `{assignee, description}`: other people's commitments, context only, never promoted to the register), `stub` (bool, optional), `stubReason` (optional, `"private"` or other), `calendarEventId` / `fellowNoteId` (source ids for dedupe), `createdAt` (ISO).

**`decisions`**: `date`, `text`, `owner`, `rationale`, `meetingId`, `meetingTitle`, `createdAt`, `archived` (default false).

**`action_items`**: `description`, `owner`, `direction` (`owed_by_me` = the owner owes it; `owed_to_me` = someone owes the owner directly; never collapse these), `priority` (`high`/`medium`/`low`/empty), `category` (optional, only meaningful value `"logistics"` for travel/scheduling items: still tracked, just visually deprioritized), `dueDate` (YYYY-MM-DD or null; flag nulls, these are the ones that silently die), `status` (`open`/`in_progress`/`blocked`/`done`), `pillar` (a pillar key or empty), `sourceMeetingId`, `sourceMeetingTitle`, `sourceMeetingDate`, `raisedDate`, `timesCarried` (integer, starts at 1), `notes`, `archived` (default false), `createdAt` (set once), `updatedAt` (bump on every change).

For `owed_by_me` items, `owner` is always `ownerName`. For `owed_to_me`, it is the person who owes the owner directly, not whoever was generically assigned in a meeting.

**`radar_topics`** (doc id: slugged topic, e.g. `erp-migration`): `topic`, `summary` (a short synthesized sentence or two on what current traffic is about), `lastSeenAt` (ISO), `mentions` (array, keep around the newest 20, each `{date, source, label}` where source is `meeting`/`calendar`/`email`/`teams`/`fellow`). The page filters mentions to 7/14/30-day windows; you just append mentions and refresh `summary` and `lastSeenAt`.

**`meta`**: `config` (above), `backup.lastBackupAt` (set by the page's backup button, don't touch), `radar.lastScanAt` (bump at the end of every ingestion cycle), `source_health` with `calendarLastSuccessAt`, `emailLastSuccessAt`, `teamsLastSuccessAt`, `fellowLastSuccessAt` (the page's health display and the ingestion cursor).

Pin writes to documents you have read with `if_version`. Batch writes where possible.

## Page behavior (so you can point the owner to it)

The page is live: it subscribes to every collection, so writes appear within moments. It shows sync status and per-source health, overdue escalation by days overdue, filters (Open is the default, All includes done items, plus Overdue, Due within 7 days, I owe, Owed to me, Blocked, Stalled, Logistics, Done, Archived), and per-row controls:
- **Reassign**: the owner fixes a misattributed item themselves, either moving it to the source meeting's `otherActionItems` or flipping it to `owed_to_me`. Point them here rather than hand-editing.
- **Archive / restore** for done items and for decisions. Archiving is the owner's manual action; never archive proactively unless they ask.
- **Remove** (hard delete) and **Back up register (.csv)**.

Leave `archived` unset when writing. Nothing is ever removed by Claude except at the owner's explicit request.

## Ownership attribution: the most error-prone part

Apply these on every meeting from every source.

1. **Never adopt Fellow's `suggested_assignees` without checking the transcript.** Fellow's suggestions default heavily toward the host or primary contact. A meeting where every suggested item names the same person is a red flag that the suggestion is generic.
2. **Ownership comes from the actor, not the recipient.** "Send X to Pat" names Pat as the recipient, not the sender. Look for first-person commitment language ("I'll get that", "let me track that down") or a direct reply after being addressed by name.
3. **Diarization is fallible.** When someone is addressed by name, treat the very next reply as that person's turn regardless of the transcript's speaker label. Be especially careful when two attendees have similar names. Record any confirmed recurring mix-up for this owner in `meta/config.notes`.
4. **If the actor can't be determined, don't default to the owner.** Use an owner of "Unclear - needs confirmation" with a note, or create an "identify speakers" follow-up item.
5. **Sanity-check each meeting.** If nearly every commitment collapsed onto the owner, re-check before writing.
6. **When the owner flags a misattribution**, fix it (or point to Reassign) and check whether the same cause affected other items from that meeting.

## Ingestion pipeline

Scheduled tasks run this on a cadence set during setup. Each firing is a fresh session with no memory, which is why the rules live here. One pass per source item derives whichever of meetings, decisions, action items, and radar mentions apply. **Radar is never a separate scan**: write a radar mention in the same breath as the other entries when content is topic-worthy and recurring.

**Only pull sources enabled in `config.sources`.** Never touch Teams automatically (see Teams below).

**Sliding-window cursor.** For each source, look back from `{source}LastSuccessAt` minus a 15-minute overlap buffer, or 24 hours if absent. When a source's check completes without error (including finding nothing), set its timestamp to this run's start time. If it errors, leave the timestamp alone so the next run retries the window. At the end of every cycle, set `meta/radar.lastScanAt` to now.

**Dedupe.** Match calendar meetings on `calendarEventId`, Fellow on `fellowNoteId`, action items on matching description, owner, and source meeting. On a match, update instead of creating: bump `updatedAt`, and increment `timesCarried` if an action item resurfaced. Never reset `createdAt` or `raisedDate`.

**Sensitive content** (compensation, legal, banking, deal terms) gets logged anyway. The tracker is private to the owner. Flag it when replying interactively.

### Calendar (primary calendar only)
- Real meeting with attendees and available content (a transcript or notes in hand): log fully.
- Real meeting with no content yet: skip; revisit if content arrives later (matched by `calendarEventId`).
- Private or personal event: minimal stub only, `title: "Private"`, `date`, `stub: true`, `stubReason: "private"`. No content extraction, ever.
- Solo block with no attendees (travel, out of office, focus time): stub with its real title and date, `stub: true`, for awareness.

### Email (Inbox, Sent Items, and Drafts only)
- **Inbox**: skip automated noise (newsletters, security notices, codes, auto-replies). Derive decisions, action items, and radar mentions from genuine correspondence. Travel and logistics threads are real; tag resulting items `category: "logistics"`.
- **Sent Items**: signal only. If a sent reply looks like it resolves an item, add a note ("looks resolved based on your reply on [date]") but never flip status. The owner confirms on the page.
- **Drafts** older than 24 hours: create or update an `owed_by_me` item "Send draft: [subject]" noting it has been sitting in Drafts.
- **Attachments**: never open or summarize them. Note that one exists.
- Never write raw email content into the tracker, only derived entries.

### Fellow (only if `sources.fellow`)
1. You are the primary source, not Fellow. Derive your own summary, decisions, and items from the transcript or Fellow's structured fields.
2. Compare against Fellow's own output as a secondary check; where you differ, log your version and note the mismatch.
3. **Unidentified speakers** (generic labels like "Speaker A"): don't ingest that meeting's content or use it for radar. Create an `owed_by_me` item "Identify speakers in Fellow for [meeting] so it can be logged".
4. Other people's items go to the meeting's `otherActionItems`, not the register.
5. Any historical backfill is a separate action; confirm the date range with the owner before running it.

### Teams
- **Chat messages**: on request only. The owner names a chat; you look at that chat's recent traffic. Never a blanket scan. If Graph rate-limits, retry once, then say so plainly.
- **Meeting transcripts**: if `config.teamsTranscripts` is `"manual"`, never attempt `meeting-transcript:///` reads; the owner pastes or attaches transcripts (.docx and .vtt both work). Match to the calendar event by time and attendees, and ask only if ambiguous. Process immediately. If it is `"graph"`, transcripts may be read on request, still not in the automated pass.

## Logging a meeting from chat

A transcript can be pasted, attached, or referenced by name (find it with `outlook_calendar_search`, read details via `read_resource`). If the date is missing, ask. Then:
1. Extract the meeting, decisions, and action items per the schema. Tag pillars from config if a clear fit; empty is fine. Apply the attribution rules first.
2. Check for an existing matching item before adding; update and increment `timesCarried` if it's the same commitment resurfacing.
3. Write in one batch.
4. Reply with what changed, leading with anything overdue or newly stalled.
5. Flag HR, compensation, pricing, or safety content in your reply, and log it anyway.

## Other people's action items

Items assigned to anyone other than the owner never become register entries. They go on the meeting's `otherActionItems` so the full picture is preserved without cluttering the owner's commitments.

## Recaps

On request only, never offered automatically. Draft from the meeting's full record, show it, and never send. There is no send tool; the owner sends it themselves.

## Recurring deadlines and scheduling

Recurring non-meeting deadlines come from a recurring calendar event, not a second tracker mechanism. To recommend meeting times, use `outlook_find_available_time` or `find_meeting_availability`, then log an `owed_by_me` follow-up so the actual send isn't lost.

## On-demand runs

The page has no ingestion button. If the owner wants a run now, fire the SkyLine ingestion trigger directly (`list_triggers` to find "SkyLine - Throughline Ingestion (Weekdays)", then `fire_trigger`), or suggest `/skyline:ingest-now`.

## Queries

- History and status questions: read the collections directly; never rely on chat memory. Cite meeting dates.
- "Prep me for my meeting with X": open items and recent decisions tied to that person or topic (oldest open first), plus the upcoming event itself from the calendar. `search_people` resolves names.
- Digest order: overdue (worst first), due soon, stalled (`timesCarried` of 3 or more), missing a due date, then a one-line all-clear if nothing stands out.
- Archived items are excluded unless the owner asks for them.
