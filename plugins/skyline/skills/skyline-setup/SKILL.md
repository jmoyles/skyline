---
name: skyline-setup
description: "Install, upgrade, check, or remove SkyLine (Throughline tracker plus Ridgeline digest) in the current person's Claude account: publishes their pages, creates their config, and schedules their tasks. Use for /skyline:setup, /skyline:upgrade, /skyline:status, and /skyline:uninstall, or whenever someone asks to set up, repair, reconfigure, or remove SkyLine, Throughline, or Ridgeline."
---

# SkyLine setup

SkyLine has three moving parts that a plugin cannot carry, so this skill creates them inside the person's own account: two published artifacts (Throughline and Ridgeline), a config record, and scheduled tasks. All logic lives in the plugin's skills; the tasks are short pointers to them. That is what lets plugin updates reach everyone without touching their tasks.

Current plugin version: **1.0.0**.

Templates: `templates/throughline.html` in this skill's folder, and `template.html` in the ridgeline skill's folder. If you can't find them, run `find / -path "*skyline*" -name "*.html" 2>/dev/null`.

Be brisk and friendly. The person just wants it working. Ask everything in one message, with defaults they can accept by saying "defaults".

## Install (/skyline:setup)

### 1. Preconditions

- **Scheduling tools.** Confirm this session can list and create scheduled tasks (for example `list_triggers` plus a create tool). If not, stop and say: setup has to run in Claude Cowork or Claude Code, because those are the surfaces that can create scheduled tasks. Everything else works in chat afterward.
- **Microsoft 365.** Confirm Microsoft 365 tools are available (such as `outlook_calendar_search`). If not, ask them to connect the Microsoft 365 connector and rerun. Call `get_me` to prefill name, title, and email.
- **Fellow.** Note whether Fellow tools are available. Ask later whether they use it.
- **Existing install.** Call Artifact action "list" and look for artifacts titled "Throughline" or "Ridgeline", and call `list_triggers` for tasks whose names start with "SkyLine". If found, ask whether to reuse them (run Upgrade instead) or start fresh. Starting fresh leaves the old artifacts in place; they can delete them later.

### 2. One round of questions

Prefill from `get_me` and show defaults:

1. Name, title, company, email (prefilled).
2. Timezone (default: the mailbox timezone if you can read it, else ask).
3. Working hours for ingestion (default 6am to 8pm). Weekdays run hourly in that window; weekends every 3 hours in the same window. Hourly is the practical floor for scheduled tasks; don't offer faster.
4. Categories. Default: "Pillars" with People, Process, and Systems. They can accept, rename, replace with up to 6 of their own, or turn categories off.
5. Do they use Fellow for meeting notes? (Only ask if Fellow tools are present; if not present but they say yes, tell them to connect the Fellow connector and rerun.)
6. Ridgeline digest time (default 7:00am daily, including weekends) and weekly rollup (default Mondays 8:00am).
7. Also schedule the Morning Brief? Only offer this if a skill named `morning` is available in the session; default no, since Ridgeline covers most of the same ground.

### 3. Publish Throughline

1. Call Artifact action "capabilities" first. The page uses the `db` and `downloads` capabilities; declare both in the form the result specifies.
2. Copy `templates/throughline.html` to `/mnt/user-data/outputs/skyline/throughline.html` and publish it: title "Throughline", favicon 🧵, with those capabilities. Record the URL.
3. Initialize the database with one `write_db` batch (no `if_version`, these are new):
   - `meta/config`: `ownerName`, `ownerTitle`, `company`, `email`, `timezone`, `pillarWord`, `pillars` (array of `{key, label}`, keys lowercase slugs), `sources` (`{calendar: true, email: true, fellow: <answer>}`), `teamsTranscripts: "manual"`, `throughlineUrl`, `installedVersion: "1.0.0"`, `installedAt` (ISO), `notes: []`.
   - `meta/source_health`: `{}`
   - `meta/radar`: `{}`
   - `meta/backup`: `{}`

### 4. Publish Ridgeline

Copy the Ridgeline template, fill `[OWNER NAME]` and `[COMPANY]`, set today's date in the masthead, dashes in the four stat tiles, and replace the section-stack comment with `<div class="allclear">Your first digest arrives [day] at [time].</div>`. Publish with title "Ridgeline", favicon ⛰️ (no capabilities needed). Then `write_db` update `meta/config` with `ridgelineUrl`.

### 5. Schedule the tasks

**Timezone first.** Prefer timezone-aware cron: prefix the expression with `CRON_TZ=<IANA zone> `. Create the first task that way, then check its next run time in `list_triggers`. If the scheduler rejected the prefix, or the next run doesn't land at the intended local time, delete it and fall back to UTC: convert each hour using the zone's current offset, and also create two one-shot reminder tasks named "SkyLine - DST adjustment" that fire the day after the next daylight-saving change, with a prompt to shift every SkyLine task's hours by the change and then schedule the next reminder. Tell the person which mode you used.

Pick one minute past the hour between 5 and 55 at random and use it for all ingestion runs, so installs don't all fire on the hour.

With start hour S and end hour E (local, E exclusive), create:

| Name | Cron (local) | Connectors |
|---|---|---|
| SkyLine - Throughline Ingestion (Weekdays) | `M S..E-1 * * 1-5` (hour list) | Microsoft 365, plus Fellow if enabled |
| SkyLine - Throughline Ingestion (Weekends) | `M S,S+3,... (below E) * * 0,6` | Microsoft 365, plus Fellow if enabled |
| SkyLine - Ridgeline Daily Digest | digest time, `* * *` | Microsoft 365 |
| SkyLine - Throughline Weekly Rollup | rollup time and day | Microsoft 365 |
| SkyLine - Morning Brief (optional) | digest time plus 1 hour, weekdays | Microsoft 365 |

Email notifications on, push off, for every task. Use the prompts below verbatim, filling in the URLs.

**Ingestion prompt (both weekday and weekend):**

```
You are running one firing of the SkyLine Throughline ingestion pipeline. Throughline URL: {THROUGHLINE_URL}

Load the throughline skill (Skill tool) and read it in full. It is the authoritative reference; this prompt only points to it. Then read meta/config and meta/source_health from that artifact and run the skill's "Ingestion pipeline" section exactly, pulling only the sources enabled in config. Do not touch Teams automatically. Write via write_db, batching where possible. On each source's success, set its source_health timestamp; on error, leave it alone. Set meta/radar.lastScanAt to now at the end.

This is an unattended background run. Do not ask questions; make the reasonable call per the skill. End with a short factual summary of what was written, or "nothing new this cycle".
```

**Ridgeline prompt:**

```
Run the SkyLine Ridgeline daily digest. Throughline URL: {THROUGHLINE_URL} Ridgeline URL: {RIDGELINE_URL}

Load the ridgeline skill (Skill tool) and follow it exactly, including publishing in place to the Ridgeline URL and delivering the email with PushNotification. This is an unattended run; do not ask questions.
```

**Weekly rollup prompt:**

```
Run the SkyLine Throughline weekly rollup. Throughline URL: {THROUGHLINE_URL}

Load the throughline-rollup skill (Skill tool) and follow it exactly, including delivering it with PushNotification. This is an unattended run; do not ask questions.
```

**Morning Brief prompt (optional):**

```
/morning
This is an unattended scheduled run with no one present, so skip connector suggestions and render the brief per the skill's ground rules for unattended runs.
```

### 6. Smoke test

1. Fire the weekday ingestion task now with `fire_trigger`. When it finishes, re-read `meta/source_health` and confirm a timestamp for each enabled source. If a source is missing, report the likely cause (connector not signed in, or Fellow not connected) rather than claiming success.
2. Fire the Ridgeline task and confirm the Ridgeline page updated.
3. Ask them to check their inbox for the Ridgeline email in the next few minutes. If it doesn't arrive, check that email notifications are on for the task.

### 7. Wrap up

Reply briefly with: both links (tell them to bookmark Throughline), the schedule in their local time, and how to use it: paste or attach any transcript in chat to log it, ask "what's overdue" or "prep me for my meeting with X", and run `/skyline:ingest-now` for an immediate pull. Mention Teams transcripts are manual by default: paste or attach them.

## Upgrade (/skyline:upgrade)

1. Find the Throughline artifact and read `meta/config`. If `installedVersion` equals the current plugin version, say they're current and stop.
2. Read the Throughline artifact (Artifact action "read"), then republish `templates/throughline.html` to its URL. This replaces the page only; the database is untouched. Keep the same favicon and capabilities, and check action "capabilities" in case the declaration format changed.
3. Compare each "SkyLine" task's prompt and connectors to the templates above. Editing a task replaces its prompt wholesale, so rewrite it in full with the current template and the existing URLs.
4. Add any new `meta/config` fields with their defaults, never overwriting existing values, and set `installedVersion`.
5. Summarize what changed.

## Status (/skyline:status)

Report, without changing anything: installed version versus current, both links, each SkyLine task's schedule in local time and whether it's enabled, and each source's last success time from `meta/source_health`, flagging anything older than 2 hours during working hours. Suggest the fix for anything stale.

## Uninstall (/skyline:uninstall)

Confirm first. Then delete every task whose name starts with "SkyLine". Leave the artifacts in place and tell them they can delete the Throughline and Ridgeline artifacts themselves if they want the data gone; removing the plugin itself is done in Customize, Plugins.
