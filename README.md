# Email-to-Calendar Sync — a Claude Code skill

A [Claude Code](https://claude.com/claude-code) skill that runs as a daily scheduled task: it watches your Gmail for confirmation emails from a service you care about, parses out the structured details, and mirrors each new event to your Google Calendar — without ever creating a duplicate.

It's a small, focused agent that solves a real annoyance: you book something (a Pilates class, a restaurant reservation, a flight, a delivery) and now have to manually transcribe it onto your calendar. This skill closes that loop autonomously.

## What it does

```
┌──────────────────┐      ┌──────────────┐     ┌─────────────────┐     ┌────────────────┐
│  Scheduled Task  │ ───► │  Gmail MCP   │ ──► │   Extraction    │ ──► │  list_events   │
│   (daily, e.g.   │      │  (read-only) │     │  (judgment +    │     │  (dedupe       │
│    8 AM ET)      │      │              │     │   plain-text    │     │   check)       │
└──────────────────┘      └──────────────┘     │   hints)        │     └────────┬───────┘
                                               └─────────────────┘              │
                                                                                 ▼
                                                                        ┌────────────────┐
                                                                        │  create_event  │
                                                                        │  (only if not  │
                                                                        │   duplicate)   │
                                                                        └────────────────┘
```

Run on a schedule (e.g. once a day, or once an hour during high-booking periods), the agent:

1. Searches Gmail for confirmation emails matching the patterns you configured.
2. Extracts the event title, start time, and (optional) location from each match.
3. Checks the target calendar for that day to make sure the event isn't already there.
4. Creates the event only if it's new — suffixed with the service name so mirrored events are visually distinct.
5. Logs the run quietly. Notifies you only if something breaks.

## Use cases

The skill is service-agnostic. Configure it for whatever you want auto-mirrored:

- **Fitness classes** — Pilates, yoga, Barre, climbing, group runs
- **Restaurant reservations** — Resy, OpenTable, Tock confirmations
- **Flights and trains** — airline / rail confirmation emails
- **Deliveries** — "arriving today" notifications from carriers
- **Doctor / dentist / vet appointments** — confirmation emails from booking systems
- **Online events** — webinar, conference, or workshop registrations

Anything that lands as a structured email becomes a structured calendar entry, no manual transcription.

## Repo layout

```
email-to-calendar-skill/
├── README.md
└── scheduled-tasks/
    └── email-to-calendar-sync/
        └── SKILL.md      ← the recipe + inline config block
```

## Setup

1. **Install Claude Code** and connect the **Gmail** and **Google Calendar** MCP servers.
2. **Drop the skill** into your scheduled-tasks directory:
   ```
   ~/.claude/scheduled-tasks/email-to-calendar-sync/SKILL.md
   ```
   Or copy and rename per service (e.g. `pilates-sync`, `flights-sync`, `resy-sync`) — see "One service per skill instance" below.
3. **Personalize the config block** at the top of `SKILL.md`:
   - `service_name`, `calendar_id`, `timezone`
   - `gmail_queries` — sender + subject patterns specific to your service
   - `extraction` hints — plain-English hints telling the agent where to find the title, start time, and location in the body
4. **Register it as a scheduled task** in Claude Code — daily at your preferred time. Once-a-day in the morning is usually enough; bump to hourly if you book things often.
5. **Verify the first run** by checking the target calendar for the expected events (and that re-running creates no duplicates).

## Design choices worth calling out

- **Idempotent by pre-check, not by stored state.** Instead of maintaining a "processed thread IDs" file, the agent asks the calendar before writing: *"is this event already here?"* — simpler, self-correcting, and survives the agent being run from a fresh machine. If the calendar is wiped, the next run rebuilds it from email; if email is wiped, the calendar still reflects what was already mirrored. There's no third source of truth to drift.
- **Service name as title suffix.** Mirrored events are tagged (e.g. `Yin & Sound — Example Studio`) so they're visually distinct from manually-created ones, and trivial to bulk-filter or bulk-delete if the sync ever misbehaves.
- **One service per skill instance.** Multiplexing services inside one config makes failure modes hard to reason about. Copy the skill per service, configure each independently, schedule them on whatever cadence each one needs.
- **Plain-language extraction hints.** The config doesn't ask you to write regexes — you describe where to find each field in plain English ("the first date+time in the body"). The agent uses judgment, which handles slight format variations gracefully.
- **Quiet by default.** Successful runs produce no notification — only failures do. Notifications about routine success train you to ignore them, which is exactly when you stop noticing the real failures.
- **Read-only on Gmail.** The skill never archives, labels, or marks emails as read. The inbox stays the source of truth; the calendar is a derived view.

## What this demonstrates

For an AI/agent product portfolio, this skill is small but rich in pattern:

- **Autonomous scheduled execution** — fire-and-forget, no human in the loop
- **Multi-MCP orchestration** — Gmail (read) + Google Calendar (read + write)
- **Structured extraction from semi-structured input** — emails follow loose conventions, not schemas
- **Idempotent writes** — the "check before write" pattern that keeps repeated runs safe
- **Configuration vs. logic separation** — the recipe is one file; service-specific knobs sit in a YAML block at the top

## License

MIT.
