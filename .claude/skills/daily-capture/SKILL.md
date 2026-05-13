---
name: daily-capture
description: Captures and organizes daily activity into chronological daily logs and long-running topic/project logs. Run with /daily-capture. Pulls from Google Calendar, Gmail sent mail, and chat history, then routes extracted content into configured workspace folders such as projects, school, research, career, music, and personal. Also accepts optional voice transcript or free-form text input.
allowed-tools: Bash, Read, Write, mcp__google_calendar, mcp__gmail
disable-model-invocation: false
---

# Daily Capture Skill

You are a personal knowledge manager. When this skill runs, you reconstruct the user's day from their digital footprint, merge in any optional manual input, preview the plan, then write structured notes into the knowledge base.

---

## Workspace Layout

This skill assumes a knowledge base with two kinds of notes:

1. **Chronological notes** — daily logs organized by date.
2. **Long-running topic notes** — project, course, research, career, music, or personal folders that collect factual updates over time via `_log.md` files.

Default folder layout:

```text
daily/
projects/
school/
research/
career/
music/
personal/
templates/
00-inbox/
_index.md
```

Only use folders that actually exist. Never create new folders.

### Daily log file

Daily logs are written to:

```text
daily/YYYY/MM-month/YYYY-MM-DD.md
```

Example: `daily/2026/05-may/2026-05-12.md`

If the month folder does not exist, create it.

### Topic activity files

Long-running topic updates are appended to `_log.md` inside the relevant folder:

```text
[path-to-topic]/_log.md
```

Examples:
```text
projects/noahs-ark-music-eval/_log.md
projects/cerebras-thesis/_log.md
school/spring-2026/cse-446/_log.md
career/fellowships-residencies/_log.md
music/awaaz/_log.md
```

Do not append daily updates into `_index.md`. `_index.md` is navigation only. `_log.md` is dated activity.

If `_log.md` does not exist in a routable folder, create it with:

```markdown
# Activity Log

Dated factual updates routed here by daily-capture.

---
```

### Root index file

```text
_index.md
```

Updated every run with links to recent daily logs and known topic folders.

---

## Day Boundary

A "day" runs from **5:00am to 4:59am the following morning**.
The file is always named by the date the day *started*.
Example: activity at 1:00am on May 13 belongs to the May 12 file.

---

## Step 1 — Compute log date and check for existing file

**Before anything else**, run this exact command to compute the correct log date and path:

```bash
python3 - <<'PY'
from datetime import datetime, timedelta
now = datetime.now()
d = now.date() if now.hour >= 5 else now.date() - timedelta(days=1)
month_folder = d.strftime("%m-%B").lower()
print(d.strftime("%Y-%m-%d"))
print(d.strftime("%Y"))
print(month_folder)
PY
```

Use the output to set:
- `LOG_DATE` = line 1 (e.g. `2026-05-12`)
- `LOG_YEAR` = line 2 (e.g. `2026`)
- `LOG_MONTH_FOLDER` = line 3 (e.g. `05-may`)
- `DAILY_PATH` = `daily/LOG_YEAR/LOG_MONTH_FOLDER/LOG_DATE.md`

**Never use the system date directly. Always go through this computation first.**

The calendar and Gmail pull windows must be `LOG_DATE 05:00:00` → `LOG_DATE+1 05:00:00`.

Then check if the file already exists:

```bash
ls "DAILY_PATH"
```

- If it exists and `--auto` flag was passed → print "Daily log for LOG_DATE already exists. Skipping." and exit.
- If it exists and no `--auto` flag → silently proceed to append an update block.

---

## Step 2 — Scan workspace structure

Scan for routable topic folders using:

```bash
find projects school research career music personal -type d 2>/dev/null | sort
```

A folder is routable if it contains `_index.md`, contains `_log.md`, or is a meaningful named folder under `projects/`, `school/`, `research/`, `career/`, `music/`, or `personal/`.

Never route to top-level category folders themselves (`projects/`, `school/`, etc.) unless they have a `_log.md`.

Also check and create the month folder if needed:

```bash
ls daily/LOG_YEAR/
```

---

## Step 3 — Pull from data sources

Pull all three sources for the day window (5am–4:59am). Be explicit about what you found and what you didn't.

### 3a. Google Calendar
List all calendars first, then pull events from every calendar in the day window.

For each event extract: calendar name, title, start time, end time, attendees.

### 3b. Gmail (sent only)
Fetch emails sent by the user during the day window.
For each email extract: subject, recipient(s), rough topic.
Do not extract body content.

### 3c. Claude chat history
Use `conversation_search` if available. Run multiple searches with topic keywords. Also run `recent_chats`.
If tool unavailable, note "0 chats found (tool unavailable)".

---

## Step 4 — Accept optional manual input

After pulling sources, ask:

```
--- Optional input ---
Paste a voice memo, quick bullets, raw notes, git log, or anything else.
Or answer any of these:

  1. What did you actually work on or do today?
  2. Did anything go well, or click into place?
  3. What's blocked, unfinished, or carrying over?
  4. Anyone you talked to or worked with worth noting?
  5. Anything personal — how you're feeling, random thoughts, goals?

(Type END on a new line when done, or just press Enter to skip)
```

Question 5 answers go to the daily log only — never routed to topic files.

---

## Step 5 — Extract and route

For each piece of content decide:

- Personal/emotional content → daily log only (Personal section), never `_log.md` files.
- Factual/actionable content → both daily log and relevant `_log.md`.
- No clear match → daily log only.

### Routing Priority

When multiple folders match, route by this priority:

1. Explicit project/course/entity match (named in content).
2. Active project folder under `projects/`.
3. Course folder under `school/`.
4. Research planning folder under `research/`.
5. Career/fellowship/application folder under `career/`.
6. Music folder under `music/`.
7. Personal folder under `personal/`.
8. Daily log only.

### Routing examples

| Content mentions | Route to |
|-----------------|----------|
| "Cerebras", "model bringup", "kernel", "eDSL" | `projects/cerebras-thesis/_log.md` |
| "Noah's ARK", "music meeting", "curator", "dataset", "score-performance" | `projects/noahs-ark-music-eval/_log.md` |
| "AI homogenization", "substrate-aware", "LLM social discovery", "insane idea" | `projects/moonshot-ideas/_log.md` |
| "CSE446", "RL homework", "ML lecture" | `school/spring-2026/cse-446/_log.md` |
| "CSE452", "distributed systems" | `school/spring-2026/cse-452/_log.md` |
| "CSE579", "optimization" | `school/spring-2026/cse-579/_log.md` |
| "PhD", "advisor", "research identity" | `research/phd-planning/_log.md` |
| "Anthropic fellowship", "OpenAI residency", "Google Student Researcher" | `career/fellowships-residencies/_log.md` |
| "Awaaz", "rehearsal", "performance" | `music/awaaz/_log.md` |
| ARK talk / guest speaker | `projects/noahs-ark-music-eval/_log.md` |

---

## Step 6 — Preview plan

Before writing anything, show:

```
=== DAILY CAPTURE PREVIEW — YYYY-MM-DD ===

SOURCES PULLED:
  ✓ Google Calendar: [N] events
  ✓ Gmail sent: [N] emails
  ✓/– Claude chat history: [N relevant chats / tool unavailable]
  ✓/– Manual input: [provided / skipped]

DAILY LOG → daily/YYYY/MM-month/YYYY-MM-DD.md
  [full draft]

TOPIC FILE UPDATES:
  → projects/[topic]/_log.md
     [bullets that will be appended]
  → school/[term]/[course]/_log.md
     [bullets that will be appended]

UNROUTED (daily log only):
  - [item] — reason

INDEX → _index.md
  [what changes]

Proceed? [Y to confirm, or describe changes]
```

Wait for confirmation before writing anything.

---

## Step 7 — Write files

### Daily log format

```markdown
# YYYY-MM-DD

## Events
- HH:MM – HH:MM | [Event title] | [Calendar] | [Attendees if any]

## Emails Sent
- HH:MM | [Subject] → [Recipient(s)]

## Chats / Work Sessions
- HH:MM | [Topic worked on]

## Notes
- [HH:MM] [Factual note]

## Routed Updates
- `projects/example/_log.md` — [short summary]
- `school/spring-2026/cse-446/_log.md` — [short summary]

## Carryover
- [ ] [Unfinished task or open question]

## Personal
- [Personal reflections, mood, thoughts — stays here only]
```

If appending (file already exists), add:

```markdown
---
### Update — HH:MM

[new content block]
```

### Topic file format (`_log.md`, append only)

```markdown
### YYYY-MM-DD

- [Factual update]
- [Decision, thing learned, meeting outcome, or task completed]
```

No timestamps. No personal content. No emotional content.

### Index format

Rewrite `_index.md` completely each run:

```markdown
# Knowledge Base Index
_Last updated: YYYY-MM-DD_

## Recent Daily Logs

| Date | File |
|------|------|
| YYYY-MM-DD | [path](path) |

## Active Projects

| Project | Log |
|---------|-----|
| [name] | [_log.md](path) |

## School

| Course | Log |
|--------|-----|
| [name] | [_log.md](path) |

## Research / Career / Music / Personal

| Area | Log |
|------|-----|
| [name] | [_log.md](path) |
```

---

## Step 8 — Optional sync

After writing files, print the list of files changed.

If `--sync` is passed and a git repository exists:

```bash
git status --short
```

Then ask the user before making any version-control changes, unless `--auto --sync` was explicitly passed. If `--auto --sync`, run:

```bash
git add -A
git commit -m "daily capture: YYYY-MM-DD"
git push
```

---

## Auto-run mode (`--auto`)

- Skip manual input step.
- Skip preview/confirmation — write immediately.
- Use only automatic sources.
- Check for existing file first and exit if found.
- Run sync only if `--sync` also passed.

---

## Error handling

- Calendar or Gmail MCP fails → note as "⚠ Could not pull [source]" in daily log and continue.
- Git push fails → warn but do not block.
- `_log.md` does not exist → create with standard header before appending.
- Month folder missing → create it.
