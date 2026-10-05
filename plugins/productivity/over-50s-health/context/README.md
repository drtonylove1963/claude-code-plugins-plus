# Context Files

The over-50s-health-advisor agent uses local Markdown files to maintain health context, preferences, and session
history. This approach keeps personal health information private and under the user's control.

## Architecture

### Templates (This Repository)

This repository contains template files in `context/templates/`:

- `INITIAL_USER_INFORMATION.md`
- `CLIENT_HEALTH_CONTEXT.md`
- `CLIENT_PREFERENCES.md`
- `SESSION_NOTES.md`
- `SOURCES.md`
- `SESSION_NOTES_ARCHIVE.md`
- `METRICS_LOG.csv`

These templates provide structure and examples but contain no real user data. When the plugin is installed, the
`SessionStart` hook copies them to `~/.claude/over-50s-health-advisor/templates/` before each session so the agent
can read them without needing to know the version-specific plugin cache location.

### User Context Files

On the first conversation, the agent creates personal context files at:

```text
~/.claude/over-50s-health-advisor/context/
├── INITIAL_USER_INFORMATION.md     # core — read every session
├── CLIENT_HEALTH_CONTEXT.md        # core — read every session
├── CLIENT_PREFERENCES.md           # core — read every session
├── SESSION_NOTES.md                # core — read every session (last ~2 entries only)
├── SOURCES.md                      # core — read every session
├── SESSION_NOTES_ARCHIVE.md        # history — read only on demand
└── METRICS_LOG.csv                 # analysis — read only on demand
```

These files contain actual health information and are never committed to version control.
Reinstalling or updating the plugin never touches these files. The agent checks for each file individually on
first run, so upgrading from an older install that only has the original five files simply adds the two new
ones alongside — nothing is renamed, moved, or rewritten.

## Context File Descriptions

### INITIAL_USER_INFORMATION.md

Basic demographic and goal information:

- Age, sex, primary goals
- Current activity level
- Time and equipment availability
- Initial questions or concerns

### CLIENT_HEALTH_CONTEXT.md

Medical and health history:

- Conditions, medications, surgeries
- Injuries, limitations, contraindications
- Recent lab results or trends
- Healthcare provider information

### CLIENT_PREFERENCES.md

Preferences for guidance:

- Preferred units (imperial/metric)
- Dietary preferences or restrictions
- Exercise preferences
- Communication style preferences

### SESSION_NOTES.md

Chronological log of interactions:

- Date-stamped session summaries
- Key decisions or plans made
- Progress updates
- Questions for follow-up

### SOURCES.md

Curated list of evidence-based resources:

- High-quality sources the agent has cited
- Personal research findings
- Clinician-provided resources

### SESSION_NOTES_ARCHIVE.md

Full-detail narrative for sessions older than the ~2 most recent kept in `SESSION_NOTES.md`. Not read at
session start — only when the agent needs historical detail (e.g., a clinician report question, or "why did
this change"). This keeps the active, every-session context small without losing any history: entries are
moved here, not deleted. Once the archive itself grows large, its oldest entries are condensed into a short
"Condensed earlier history" section rather than continuing to grow full-detail forever.

### METRICS_LOG.csv

Tidy, append-only time series of every quantifiable metric mentioned in a session — weight, body composition,
vitals, sleep, labs, nutrition, activity, and so on — one row per metric per date:

```csv
date,metric,value,unit,note
2026-01-23,weight,185,lb,
2026-01-23,resting_hr,58,bpm,
```

This is the file trend and annual reports read from. It exists so that a "summarize my year" request pulls
structured rows instead of re-parsing a year of narrative prose in `SESSION_NOTES_ARCHIVE.md`. Metric names are
freeform (snake_case) rather than a fixed schema, since clients track different things. If a note needs a
comma, wrap it in double quotes so the row still parses as CSV.

## Required vs Optional Context

**Required** (minimum for personalized guidance):

- `INITIAL_USER_INFORMATION.md`
- `CLIENT_PREFERENCES.md`

**Optional** but strongly recommended:

- `CLIENT_HEALTH_CONTEXT.md`
- `SESSION_NOTES.md`
- `SOURCES.md`
- `SESSION_NOTES_ARCHIVE.md`
- `METRICS_LOG.csv`

## Privacy and Data Management

- All context files are stored locally on the user's machine
- Files are in plain Markdown format (readable, editable with any text editor)
- The user has full control to view, edit, or delete any information
- No data is sent to external services except when the agent performs web searches (which do not include context files)
- The agent only accesses these files when invoked

## Context Budget Management

The agent aims to keep the five **core** files (the ones read every session) under 2,000 words combined. The
two history/analysis files, `SESSION_NOTES_ARCHIVE.md` and `METRICS_LOG.csv`, are deliberately excluded from
this budget — they are only read on demand, so their size does not cost tokens on ordinary turns. The agent
will:

- Monitor total word count across the five core files at the start of each session
- Move (not condense) older `SESSION_NOTES.md` entries into `SESSION_NOTES_ARCHIVE.md` once more than ~2 full
  entries have accumulated
- Condense the archive's oldest entries into terse one-liners once it grows past ~15 full entries, since the
  precise numbers survive independently in `METRICS_LOG.csv`
- Never prune or condense `METRICS_LOG.csv` — it is designed to grow indefinitely at low cost per row
- Request approval before pruning `INITIAL_USER_INFORMATION.md` or `CLIENT_PREFERENCES.md`

## Artifact Ingestion

Lab reports, CSV files, or other documents can be provided for the agent to analyze. The agent will:

- Ask for consent before extracting and storing summaries
- Store only relevant, minimal data in context files
- Reference the artifact and extraction date in `SESSION_NOTES.md` or `CLIENT_HEALTH_CONTEXT.md`

## Editing Context Files

Context files can be edited directly with any text editor. The agent treats edits as authoritative updates.
Common workflows:

```bash
# Add new health information
nano ~/.claude/over-50s-health-advisor/context/CLIENT_HEALTH_CONTEXT.md

# Review past sessions
cat ~/.claude/over-50s-health-advisor/context/SESSION_NOTES.md

# Update preferences
code ~/.claude/over-50s-health-advisor/context/CLIENT_PREFERENCES.md
```
