---
name: email-to-calendar-sync
description: Daily sweep — find confirmation emails for a configured service (fitness classes, reservations, flights, etc.) and mirror them to Google Calendar, skipping duplicates.
---

You are running a daily sweep that mirrors confirmation emails to a Google Calendar. Work autonomously — do not ask the user for confirmation.

## Configuration

Edit this section to point at the service(s) you want to sync. Everything else in the skill is service-agnostic and reads from these fields.

```yaml
service_name: "Example Studio"           # human label, used in event titles + logs
calendar_id: "primary"                   # which Google Calendar to write to
timezone: "America/New_York"             # IANA timezone
event_color_id: "2"                      # optional — Google Calendar color (1–11) or empty
default_duration_minutes: 60             # used when end time isn't in the email
location_default: ""                     # optional — used if email doesn't carry a location

# Gmail queries that find new confirmation emails for this service.
# Combine sender + subject patterns specific enough that you won't match marketing.
gmail_queries:
  - 'from:reservations@example.com subject:"reservation is confirmed" newer_than:2d'
  - 'from:bookings@example.com "Your spot is reserved" newer_than:2d'

# Where to extract structured fields from each matching email.
# Use plain-language hints — the agent reads the body and uses judgment.
extraction:
  event_title: "the line after 'Your spot is reserved'"
  start_datetime: "the first date+time in the body, e.g. 'April 21, 2026 09:00 AM'"
  location: "optional — a street address if present in the body"
```

## Steps

1. **Search Gmail** with each query in `gmail_queries` (in parallel). Merge results, dedupe by thread ID.

2. **For each matching email**, extract the configured fields. If the snippet alone isn't enough, fetch the full thread with `get_thread`. Field-by-field rules:
   - **Event title** — use the `extraction.event_title` hint. Suffix the title with ` — {service_name}` so it's distinguishable on the calendar (e.g. `Yin & Sound w/ Jessanya — Example Studio`).
   - **Start datetime** — parse to ISO 8601 in the configured timezone.
   - **End datetime** — if not present, default to start + `default_duration_minutes`.
   - **Location** — use the value extracted from the email if present, else fall back to `location_default`. If both are empty, omit location.

3. **Check for duplicates before creating.** Call `list_events` for the target day and skip if an event with the same title already exists at that start time. This is critical — without it, every run double-books. Match titles case-insensitively but require the start time to match within ±1 minute.

4. **Create the event** on `calendar_id` via `create_event`:
   - **Summary**: the suffixed event title from step 2.
   - **Start / End**: ISO 8601 with the configured timezone offset.
   - **Location**: from step 2, or omit if empty.
   - **ColorId**: `event_color_id` if set, else omit.
   - **Description**: `Auto-added from {service_name} confirmation email.`

5. **Log results.** If the user maintains a daily note (configurable path — see the user's CLAUDE.md or skip if not configured), append one line:
   - `- HH:MM — {service_name} sweep: added N event(s): [titles]`
   - or `- HH:MM — {service_name} sweep: no new confirmations`

   If no daily-note path is configured, skip logging silently.

6. **Stay quiet.** Do not send notifications to the user unless the sweep hits an error you can't recover from. Routine completion is silent.

## Guardrails

- **Idempotency is the contract.** This task may run multiple times per day; every run after the first must be a no-op for already-mirrored events. The duplicate check in step 3 is what enforces that — never skip it.
- **Don't fabricate.** If a field is missing from the email and has no fallback, skip that event and log it as an extraction failure rather than guessing.
- **One service per skill instance.** If you sync multiple services (e.g. one studio for classes, another for flights), copy this skill and configure each instance separately. Don't try to multiplex inside a single config.
- **Time window is short.** `newer_than:2d` covers timezone slop and the rare case where the task missed a day. Don't extend further — older confirmations should already have been processed.
- **Read-only on Gmail.** This skill never archives, marks read, or labels emails. The inbox is the source of truth; the calendar is a derived view.

## Why these choices

- **Idempotent writes via pre-check, not via storing state.** Storing "already-processed" thread IDs means another file to manage and another way to drift. Asking the calendar "is this event already here?" before writing is simpler and self-correcting.
- **Service name as a title suffix.** Makes mirrored events visually distinct from manually-created ones at a glance, and makes them trivial to filter/delete in bulk if the sync ever misbehaves.
- **Quiet by default.** A successful sync is the boring case. Notifications about routine success train the user to ignore them, which is dangerous when something actually breaks.
