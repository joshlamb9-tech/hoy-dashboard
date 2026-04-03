# Dashboard Sync — Reference File

This file tells Claude Code exactly what to do when you ask it to sync your dashboard. It is the single source of truth for the sync process.

---

## What the sync does

It pulls live data from three sources — Todoist, Google Calendar, and Gmail — then writes a clean snapshot into Supabase so your dashboard always reflects the current state of your workload and pastoral picture.

Specifically it:

- Loads all open tasks from your Todoist Year 7&8 project and all other active projects
- Loads today's calendar events from your primary Google Calendar
- Loads flagged emails (labelled @Action, or unread and marked important) from Gmail — filtering out noise like Classroom notifications and Google Docs comment digests
- Analyses those emails to surface pupil names mentioned repeatedly, overdue follow-ups, and anything with a pastoral flavour
- Writes all of that into four Supabase tables, clearing stale data first so you always have a clean current snapshot

It does not archive. Every sync replaces the previous snapshot. If you need historical records, that is a separate piece of work.

---

## Supabase reference

- **Project ID:** dlcseuejvducbsjhqvze
- **Tables used:** `dashboard_tasks`, `dashboard_events`, `dashboard_emails`, `dashboard_insights`

---

## SQL — table setup (run once)

Run this in the Supabase SQL editor to create the four tables. You only need to do this once.

```sql
-- Tasks from Todoist
create table if not exists dashboard_tasks (
  id text primary key,
  content text,
  project_name text,
  due_date date,
  priority int,         -- 1 = highest (p1), 4 = lowest (p4)
  is_overdue boolean,
  url text,
  synced_at timestamptz default now()
);

-- Calendar events for today
create table if not exists dashboard_events (
  id text primary key,
  title text,
  start_time timestamptz,
  end_time timestamptz,
  location text,
  description text,
  synced_at timestamptz default now()
);

-- Flagged emails
create table if not exists dashboard_emails (
  id text primary key,
  subject text,
  sender text,
  received_at timestamptz,
  label text,           -- e.g. '@Action', 'unread-important'
  snippet text,
  synced_at timestamptz default now()
);

-- Analysis layer: insights derived from emails
create table if not exists dashboard_insights (
  id serial primary key,
  insight_type text,    -- 'pupil_mention', 'overdue_followup', 'pastoral_concern'
  pupil_name text,
  detail text,
  source_email_ids text[],
  synced_at timestamptz default now()
);
```

---

## SQL — clear tables before each sync (run at start of every sync)

```sql
truncate table dashboard_tasks;
truncate table dashboard_events;
truncate table dashboard_emails;
truncate table dashboard_insights;
```

---

## The sync prompt — paste this exactly when requesting a sync

When you want to trigger a manual sync, open a new Claude Code session and paste this prompt:

---

```
Sync my Mowden dashboard. Follow these steps in order:

STEP 1 — TODOIST
Use the Todoist MCP to fetch all open tasks from project ID 6fj58M455m4Jq7H7 (Year 7&8 project).
Also fetch open tasks from all other active projects.
For each task, note: task ID, content, project name, due date (if any), priority level, and whether it is overdue (due date is before today).
Today's date is [INSERT TODAY'S DATE e.g. 2026-04-03].

STEP 2 — GOOGLE CALENDAR
Use the Google Calendar MCP to fetch all events from the primary calendar for today only ([INSERT DATE]).
For each event, note: event ID, title, start time, end time, location, description.

STEP 3 — GMAIL FETCH
Use the Gmail MCP to fetch:
  a) All emails labelled @Action
  b) All unread emails marked as important

Filter OUT any email where the sender domain is @google.com, or the subject contains any of: "Classroom", "Google Docs", "comment", "mentioned you", "shared a document", "invitation to edit".

For each email that passes the filter, note: message ID, subject, sender, date received, which label triggered it (@Action or unread-important), and a short snippet of the body.

STEP 4 — EMAIL ANALYSIS
Review the filtered emails and extract:
  a) Pupil names mentioned in more than one email — list the name and how many emails mention them
  b) Emails that look like follow-ups where no reply has been sent and the original is more than 5 days old
  c) Any email that contains pastoral language: words or phrases like "concerned", "worried", "struggling", "incident", "behaviour", "wellbeing", "parents have been in touch", "spoke to", "not himself/herself"

STEP 5 — WRITE TO SUPABASE
Use the Supabase MCP (project ID: dlcseuejvducbsjhqvze) and execute_sql to:

First, clear the tables:
truncate table dashboard_tasks;
truncate table dashboard_events;
truncate table dashboard_emails;
truncate table dashboard_insights;

Then insert tasks:
INSERT INTO dashboard_tasks (id, content, project_name, due_date, priority, is_overdue, url)
VALUES [one row per task — use parameterised inserts or batch the values];

Then insert events:
INSERT INTO dashboard_events (id, title, start_time, end_time, location, description)
VALUES [one row per event];

Then insert emails:
INSERT INTO dashboard_emails (id, subject, sender, received_at, label, snippet)
VALUES [one row per email];

Then insert insights:
INSERT INTO dashboard_insights (insight_type, pupil_name, detail, source_email_ids)
VALUES [one row per insight];

STEP 6 — CONFIRM
Report back:
- How many tasks were written
- How many events were written
- How many emails were written (and how many were filtered out)
- How many insights were generated
- Any errors encountered
```

---

## Automating with a cron schedule

Claude Code does not have a native /schedule skill at present. The most reliable approach for now is a simple Mac cron job that opens a Claude Code session and runs the sync prompt automatically.

### Option A — Run it manually (recommended to start)

Keep the prompt above saved here. When you want to sync, open Claude Code and paste it. Takes about 60 seconds.

### Option B — Automate with a shell script and cron

This runs the sync at a fixed time each day (e.g. 07:30 before school).

**Step 1 — Save the prompt to a file**

Save the sync prompt above to `/Users/air/Documents/mowden-y78/dashboard/sync-prompt.txt` (replace [INSERT TODAY'S DATE] with a placeholder — the script will fill it in dynamically).

**Step 2 — Create the script**

Save this as `/Users/air/Documents/mowden-y78/dashboard/run-sync.sh`:

```bash
#!/bin/bash

# Fills in today's date and runs the sync prompt via Claude Code CLI
TODAY=$(date +%Y-%m-%d)
PROMPT_FILE="/Users/air/Documents/mowden-y78/dashboard/sync-prompt.txt"
LOG="/Users/air/Documents/mowden-y78/dashboard/sync.log"

# Substitute today's date into the prompt
PROMPT=$(sed "s/\[INSERT TODAY'S DATE[^]]*\]/$TODAY/g" "$PROMPT_FILE")

# Run via Claude Code and log output
echo "[$TODAY] Sync started" >> "$LOG"
echo "$PROMPT" | claude --print >> "$LOG" 2>&1
echo "[$TODAY] Sync finished" >> "$LOG"
```

Make it executable:

```bash
chmod +x /Users/air/Documents/mowden-y78/dashboard/run-sync.sh
```

**Step 3 — Set up the cron job**

Open cron in the terminal:

```bash
crontab -e
```

Add this line to run the sync at 07:30 every weekday (Mon–Fri):

```
30 7 * * 1-5 /Users/air/Documents/mowden-y78/dashboard/run-sync.sh
```

Save and close. Cron will handle it from there.

**Note:** This requires Claude Code CLI to be installed and authenticated. It also requires the machine to be awake and connected at the scheduled time. For a school machine that sleeps, use a launchd plist instead — ask for that if you need it.

---

## Troubleshooting

| Problem | Likely cause | Fix |
|---|---|---|
| No tasks appear | Todoist MCP not connected | Check MCP config in Claude Code settings |
| No calendar events | Google Workspace MCP not authenticated | Run any Calendar command to trigger OAuth |
| Emails not fetching | Gmail MCP session expired | Re-authenticate via MCP settings |
| Supabase insert fails | Table does not exist | Run the table setup SQL above |
| Sync prompt errors on date | You forgot to replace [INSERT TODAY'S DATE] | Fill in the date before pasting |

---

*Last updated: 2026-04-02*
