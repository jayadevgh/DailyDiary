---
name: daily-capture
description: Captures and organizes your daily activity into DateBased logs and SubjectBased topic files. Run with /daily-capture. Pulls from Google Calendar, Gmail (sent and received), and Claude chat history, then routes extracted content into the right folders. Also accepts optional voice transcript or free-form text input.
allowed-tools: Bash, Read, Write, mcp__google_calendar, mcp__gmail
disable-model-invocation: false
---

# Daily Capture Skill

You are a personal knowledge manager. When this skill runs, you reconstruct the user's day from their digital footprint, merge in any optional manual input, preview the plan, then write structured notes into the DailyLog folder tree.

---

## Naming Conventions

Always follow these exactly. Never deviate.

**Daily log file:**
```
DateBased/[Month YYYY]/YYYY-MM-DD.md
```
Example: `DateBased/May 2026/2026-05-12.md`

**Subject entry files:**
Each subject folder contains one running log file named after the folder itself (lowercase, hyphenated):
```
SubjectBased/[path]/[folder-name].md
```
Examples:
- `SubjectBased/TopicA/topica.md`
- `SubjectBased/Category/Subcategory/LeafTopic/leaftopic.md`

Entries inside subject files are always **appended**, never overwritten. Each new day's content gets a header block:
```
### YYYY-MM-DD
- bullet point of what happened
- another bullet
```

**Index file:**
```
_index.md
```
At the root of DailyLog. Auto-updated every run.

---

## Day Boundary

A "day" runs from **5:00am to 4:59am the following morning**.
The file is always named by the date the day *started*.
Example: activity at 1:00am on May 13 belongs to the May 12 file.

To determine the current day:
- If current time is at or after 5:00am → today's date
- If current time is before 5:00am → yesterday's date

---

## Step 1 — Check for existing file

Before doing anything else, check if a daily file already exists for today's date window.

```bash
ls "DateBased/[Month YYYY]/YYYY-MM-DD.md"
```

- If it exists and `--auto` flag was passed (cron run) → print "Daily log for YYYY-MM-DD already exists. Skipping." and exit.
- If it exists and no `--auto` flag → silently proceed to append an update block. Do not ask for confirmation.

---

## Step 2 — Scan folder structure

Read the SubjectBased tree to find all **leaf folders** — directories that have no subdirectories inside them. These are the only valid routing destinations.

```bash
find SubjectBased -type d | while read dir; do
  [ -z "$(find "$dir" -mindepth 1 -maxdepth 1 -type d)" ] && echo "$dir"
done
```

For example, given a tree with `SubjectBased/CategoryA/LeafX` and `SubjectBased/CategoryA/LeafY`, only `LeafX` and `LeafY` would be returned — not `CategoryA`.

Never write to intermediate/container folders (e.g. `School`, `Spring26`). Never create new folders. If something doesn't fit any leaf folder, it stays in the daily log only — no separate inbox file needed.

Also check which month folder exists or needs to be created under DateBased:

```bash
ls DateBased/
```

If the current month folder doesn't exist yet, create it:
```bash
mkdir "DateBased/[Month YYYY]"
```

---

## Step 3 — Pull from data sources

Pull all three sources for the day window (5am–4:59am). Be explicit about what you found and what you didn't.

### 3a. Google Calendar
Use the Google Calendar MCP to first list all available calendars, then pull events from **every calendar** in the day window — not just the primary one. The user has multiple calendars (e.g. personal, classes, office hours, clubs) and events are spread across all of them.

For each event extract: calendar name, title, start time, end time, duration, attendees.

### 3b. Gmail (sent and received)
Use the Gmail MCP to fetch emails from the day window in two passes:

1. **Sent** — emails sent by the user (`from:jayadevgh@gmail.com in:sent`)
2. **Received** — emails received that are relevant to the user's leaf folders. Search by folder keywords (e.g. course names, project names, club names) to filter noise. Skip newsletters, notifications, and automated mail.

For each email extract: direction (sent/received), subject, other party, rough topic inferred from subject.
Do not extract email body content — subject line is enough.
Route received emails to matching leaf folders the same way as any other content.

### 3c. Claude chat history
Use the `conversation_search` tool to find chats from the day window. Run multiple searches with different keywords to get broad coverage — search by topic areas like the user's known subjects, projects, and general terms like "code", "debug", "write", "research". Also run `recent_chats` to catch anything the keyword search might miss.

For each relevant chat extract: the topic of what the user was working on or asking about.
Ignore chats that are purely conversational or unrelated to work/school/projects.
If no results are found, note it as "0 chats found" rather than "tool unavailable".

---

## Step 4 — Accept optional manual input

After pulling the automatic sources, ask:

```
--- Optional input ---
Paste a voice memo, quick bullets, raw notes, git log, or anything else.
Or answer any of these prompts out loud and paste the transcript:

  1. What did you actually work on or do today?
  2. Did anything go well, or click into place?
  3. What's blocked, unfinished, or carrying over?
  4. Anyone you talked to or worked with worth noting?
  5. Anything personal — how you're feeling, random thoughts, goals?

(Type END on a new line when done, or just press Enter to skip)
```

Question 5 answers go to the daily log only — never routed to subject files.

---

## Step 5 — Extract and route

Synthesize all sources into structured extractions. For each piece of content decide:

- Which leaf folder(s) does this belong to? Match against the scanned leaf folder list.
- Is this personal/emotional content? → Daily log only, never subject files.
- Is this factual/actionable/reference-worthy? → Both daily log and relevant leaf subject file(s).
- Does it not clearly fit any leaf folder? → Daily log only, no separate routing needed.

Routing confidence rules:
- Clear match (e.g. mentions "CSE446", "Cerebras", a known project name) → route to that leaf
- Possible match but uncertain → route to both candidates with a note
- No match → daily log only, noted in the Notes section as unrouted

---

## Step 6 — Preview plan

Before writing anything, show the user a full preview:

```
=== DAILY CAPTURE PREVIEW — YYYY-MM-DD ===

SOURCES PULLED:
  ✓ Google Calendar: [N] events
  ✓ Gmail sent: [N] emails
  ✓ Gmail received (relevant): [N] emails
  ✓ Claude chat history: [N] relevant chats
  [✓/–] Manual input: [provided / skipped]

DAILY LOG → DateBased/[Month YYYY]/YYYY-MM-DD.md
  [full draft of what will be written]

SUBJECT FILE UPDATES:
  → SubjectBased/[LeafFolder]/[leafolder].md
     [bullet points that will be appended]
  → SubjectBased/[Category]/[Subcategory]/[LeafFolder]/[leaffolder].md
     [bullet points that will be appended]
  [etc.]

UNROUTED (stays in daily log only):
  - [item] — no matching leaf folder found

INDEX → _index.md
  [list of files that will be updated]

Proceed? [Y to confirm, or describe any changes]
```

Wait for user confirmation before writing anything. If the user asks for changes, adjust and show the preview again.

---

## Step 7 — Write files

After confirmation, write in this order:

### Daily log format

Each entry in the daily log gets a timestamp so you can see what time of day things happened. Use `HH:MM` 24-hour format.

```markdown
# YYYY-MM-DD

## Events
- HH:MM – HH:MM | [Event title] | [Attendees if any]

## Emails
- HH:MM | ↑ [Subject] → [Recipient]
- HH:MM | ↓ [Subject] ← [Sender]

## Claude Chats
- HH:MM | [Topic of what was worked on]

## Notes
[HH:MM] [Synthesized bullet or narrative from manual input — factual content]
[HH:MM] [Another item]

## Personal
[HH:MM] [Q5 content, personal thoughts — this section only ever appears in this file]
```

If the exact time isn't available from a source (e.g. voice memo), omit the timestamp for that item rather than guessing.

If the skill is run a second time in the same day (appending), add:

```markdown
---
### Update — HH:MM

[new timestamped content block]
```

### Subject file format (append only)

No timestamps here — subject files are clean reference logs, not timelines. Just the date header and factual bullets.

```markdown
### YYYY-MM-DD
- [factual bullet extracted from today's activity]
- [another bullet]
```

Never include personal/emotional content in subject files.

### Index format

Rewrite `_index.md` completely each run:

```markdown
# DailyLog Index
_Last updated: YYYY-MM-DD_

## DateBased
| File | Created |
|------|---------|
| [YYYY-MM-DD](DateBased/Month YYYY/YYYY-MM-DD.md) | YYYY-MM-DD |
[... most recent 30 entries ...]

## SubjectBased
| Subject | File | Last Updated |
|---------|------|-------------|
| [LeafFolder] | [leaffolder.md](SubjectBased/[path]/[leaffolder].md) | YYYY-MM-DD |
[... all leaf subject files that exist ...]
```

---

## Step 8 — Git commit

After all files are written, run:

```bash
git add -A
git commit -m "daily capture: YYYY-MM-DD"
git push
```

If the push fails (e.g. no remote set up yet), print a warning but do not fail the skill — the local write still succeeded.

---

## Auto-run mode

When invoked with `--auto` flag (from cron):
- Skip the optional manual input step entirely
- Skip the preview/confirmation step — write immediately
- Use only automatic sources (Calendar, Gmail, Claude chats)
- Still check for existing file first and exit if found
- Still git commit at the end

---

## Error handling

- If a Google Calendar or Gmail MCP call fails → note it in the daily log under the relevant section as "⚠ Could not pull [source]" and continue with what's available
- If git push fails → warn but do not block
- If a subject folder file doesn't exist yet → create it with a simple header before appending:

```markdown
# [Folder Name]
_Notes routed here by daily-capture skill_

---
```
