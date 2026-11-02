# Over-50s Health Advisor

A Claude Code Agent definition for evidence-based, age-appropriate health, fitness, nutrition, and longevity guidance
for adults 50+. The agent works with local context files only and always recommends confirming advice with a healthcare
professional.

## Features

- Evidence-based guidance with citations and a Sources section, weighted toward clinical measurements over
  device streams
- Noise-floor discipline: reports a metric as unchanged, not a trend, when its change is within measurement
  error
- Collection-cost transparency: states what decision a requested metric or artifact could change, and names
  an exit condition for any recurring log
- Optional long-term memory via [Hindsight](https://github.com/vectorize-io/hindsight) — a self-hosted recall
  and reflection layer over your local history files
- Safety boundaries and red-flag referral policy
- Local context management via Markdown files
- Install via Claude Code plugin system (`/plugin install over-50s-health@ali5ter`)
- Automatic context file creation on first run

## Repository structure

This repository contains the agent definition and context templates for distribution. When installed via the plugin
system, the agent is managed by Claude Code and your personal context files are stored in your home directory.

```text
agents/
  advisor.md                       # Agent definition (source)
context/
  templates/                       # Reference context templates
    INITIAL_USER_INFORMATION.md
    CLIENT_HEALTH_CONTEXT.md
    CLIENT_PREFERENCES.md
    SESSION_NOTES.md
    SOURCES.md
    SESSION_NOTES_ARCHIVE.md       # On-demand narrative history
    METRICS_LOG.csv                # On-demand structured metrics for trend reports
    history/                       # Per-session history templates
      INDEX.md
  README.md
.claude-plugin/
  plugin.json                      # Plugin manifest
hooks/
  hooks.json                       # SessionStart, PreToolUse, SubagentStop + Stop hooks
hooks-handlers/
  sync-templates.sh                # Copies templates from plugin cache to ~/.claude/over-50s-health-advisor/templates/
  guard-bash-scope.sh              # Agent-scoped PreToolUse hook: scopes the advisor's Bash access
  retain-session.sh                # SubagentStop + Stop hook: persists session to Hindsight via hindsight-retain.py
  check-retain-staleness.sh        # SubagentStop + Stop hook: warns when on-disk history is not yet retained
scripts/
  hindsight-retain.py              # MCP transport: retains documents into the health bank (idempotent manifest)
  backfill-history.sh              # One-off: ingest all history into Hindsight
migrate                            # Migration script for v2.x users
README.md
LICENSE
```

After installation, your personal context files are stored at:

```text
~/.claude/over-50s-health-advisor/
    context/                       # Your personal context files (auto-created on first run)
        ├── INITIAL_USER_INFORMATION.md   # core — read every session
        ├── CLIENT_HEALTH_CONTEXT.md      # core — read every session
        ├── CLIENT_PREFERENCES.md         # core — read every session
        ├── SESSION_NOTES.md              # core — read every session (last ~2 entries)
        ├── SOURCES.md                    # core — read every session
        ├── INDEX.md                      # session index — read on demand
        ├── history/                      # per-session history files
        │   └── YYYY/YYYY-MM-DD.md       # one file per session
        ├── METRICS_LOG.csv               # analysis — read on demand
        └── _legacy/                      # pre-v4.0.0 archives (never deleted)
```

## Requirements

- Claude Code CLI v2.0.73 or later
- Uses Claude Opus 4.6 by default for highest reasoning quality; override with `/model` if preferred

## Install

Inside Claude Code, run:

```text
/plugin marketplace add ali5ter/claude-plugins
/plugin install over-50s-health@ali5ter
```

The first time you start a health conversation, the agent automatically creates your context files at
`~/.claude/over-50s-health-advisor/context/`.

## Uninstall

Inside Claude Code, run:

```text
/plugin uninstall over-50s-health@ali5ter
```

This removes the plugin and agent. Your personal context files are **not** removed. To delete them:

```bash
rm -rf ~/.claude/over-50s-health-advisor
```

## Migrating from v2.x

If you previously installed via `./install.sh`, run the migration script from this repo:

```bash
./migrate
```

This removes the old manually-installed agent file. Your context files are preserved. Then install via the plugin
commands above.

## Usage

After installation, context files are automatically created at `~/.claude/over-50s-health-advisor/context/` on your
first conversation.

1. Fill in `~/.claude/over-50s-health-advisor/context/INITIAL_USER_INFORMATION.md` and `CLIENT_PREFERENCES.md`
   with your information.
2. Keep `CLIENT_HEALTH_CONTEXT.md` and `SESSION_NOTES.md` current as new information is shared with the agent.
3. Maintain `SOURCES.md` as a curated reference list (the agent will add sources; you can remove low-quality ones
   and add high-quality evidence).
4. Use the agent from any directory in Claude Code. The agent will read and update these context files automatically.
5. Keep the "Last updated" dates accurate in each file.
6. Ask for a "trend summary" or "how have I done this year" and the agent will read `METRICS_LOG.csv` — a
   structured, append-only record of every metric it has logged — instead of re-reading a year of session
   notes. See `context/README.md` for details on `METRICS_LOG.csv` and `SESSION_NOTES_ARCHIVE.md`.

## Bash access

The agent has `Bash` for local analysis: reading `METRICS_LOG.csv` and running your own health export scripts. A
`PreToolUse` hook (`hooks-handlers/guard-bash-scope.sh`, registered in `hooks/hooks.json`) scopes what it can
actually run: destructive commands (`rm`, `mv`, `sudo`, `chmod`, `chown`, `dd`, force-pushes) and network egress
(`curl`, `wget`, `nc`) are denied outright — everything else it allows outright or leaves to the normal permission
prompt.

Claude Code silently ignores `hooks:` set in a plugin agent's own frontmatter, so this hook is registered in the
plugin's `hooks/hooks.json` instead, which applies plugin-wide rather than to one agent. It self-scopes by
checking the `agent_type` field Claude Code passes it and is a no-op outside an active
`over-50s-health:advisor` session — it has no effect on your other Claude Code sessions.

## Long-term memory (Hindsight)

The advisor can use [Hindsight](https://github.com/vectorize-io/hindsight) as an optional recall and reflection
layer over your local history files. Hindsight must be self-hosted — it is **not** bundled in this plugin.

### Prerequisites

- A machine running Hindsight 0.10.3+ with an MCP endpoint
- Ollama (or another LLM server) for the LLM provider
- The `health` bank configured in Hindsight

### Setup

1. **Install Hindsight** on your host machine. See
   [Hindsight docs](https://github.com/vectorize-io/hindsight) for installation instructions.

2. **Configure the MCP server** in your Claude Code user scope:

   ```bash
   claude mcp add --scope user hindsight-health \
     http://192.168.1.63:8888/mcp \
     --env HINDSIGHT_API_KEY=your-api-key-here
   ```

   Replace the URL and API key with your actual values. The MCP server is restricted server-side to
   `recall`, `reflect`, and `retain` tools only.

3. **Enable retention** by writing the enable gate to `~/.hindsight/retain.env` — the same
   persistent-file pattern as the API key, set once and independent of how Claude Code is launched:

   ```bash
   echo 'export HINDSIGHT_RETAIN_ENABLED=true' > ~/.hindsight/retain.env
   ```

   The `SubagentStop`/`Stop` hook (`hooks-handlers/retain-session.sh`) sources that file and, when
   enabled, delegates to `scripts/hindsight-retain.py`, which retains the session's history file into
   the `health` bank over the MCP transport. It fails silently if Hindsight is unreachable. A sibling
   hook, `check-retain-staleness.sh`, warns on session end when on-disk history has not yet been
   retained (see **Checking staleness** below); it is offline and makes no network call.

4. **Backfill existing history** (one-time):

   ```bash
   ./scripts/backfill-history.sh
   ```

   This ingests all `history/` files, `CONTEXT_ARCHIVE.md`, `_legacy/` archives, and
   `METRICS_LOG.csv` rows into the Hindsight `health` bank. Retention runs over the MCP transport
   (`scripts/hindsight-retain.py`) and keeps a local idempotency manifest at
   `~/.hindsight/retain-manifest.json`; a first run seeds it from the bank, so re-runs skip what is
   already present and add nothing new.

   Options:
   - `--host HOST` — custom Hindsight host (default: 192.168.1.63)
   - `--dry-run` — show what would be retained without sending
   - `--skip-legacy` — skip `_legacy/` archives

### How it works

- **Session start**: the agent orients with a **targeted `recall`** (small budget, recent window),
  treating the result as a lead to cross-check against the core files — not an unbounded `reflect`,
  which on local models can return confident but wrong facts. If Hindsight is unavailable, it falls
  back to reading core files + the two most recent history entries.
- **Retrieval**: when the Client mentions past topics, the agent calls `recall` first and notes what
  it checked in a **History checked:** line. If the MCP call fails, it falls back to INDEX.md + Grep.
- **Session end**: the `SubagentStop`/`Stop` hooks delegate to `hindsight-retain.py`, which retains the
  session file into the `health` bank over the MCP transport (skipping it if the manifest already has
  it). If unreachable, the file on disk is untouched. A sibling hook, `check-retain-staleness.sh`,
  warns on end when on-disk history is not yet retained.

**Files are always canonical.** Hindsight is a layer on top — if it is unavailable, the agent degrades
to the plain-file behaviour with no data loss.

### Checking staleness

Because retention can miss sessions or metric groups (a session end that never fired, a backfill run
before new sessions existed, a manifest not yet re-seeded), the `check-retain-staleness.sh` hook makes
that gap observable: on every `SubagentStop`/`Stop` it compares the local manifest against the on-disk
`history/` files and `METRICS_LOG.csv` and warns (on stderr, non-blocking) when anything is not yet
retained. It is **offline** — it reads only the manifest and the local files, making no network call —
so it is safe on every session end. Run it on demand for the full list:

```bash
./hooks-handlers/check-retain-staleness.sh --report
```

The fix for any reported gap is idempotent — re-run `./scripts/backfill-history.sh`, which skips what
is already retained and adds only what is missing. The check runs only when `HINDSIGHT_RETAIN_ENABLED=true`.

### Known limitations

- `reflect` on local models (e.g., `qwen3.8:27b` via Ollama) can produce confident factual errors.
  v5.1.0 mitigates this at session start by orienting with a **targeted `recall`** instead and treating
  the result as a lead to cross-check against the files; reserve `reflect` for genuine cross-session
  synthesis, and always verify figures against the files.
- Recall quality: temporal queries are strong (8/8 in testing), entity queries are good but not perfect
  (9/12 at top-5 in testing).
- The spike results are documented in the
  [sysadmin-studio repo](https://github.com/ali5ter/sysadmin-studio/blob/main/docs/hindsight-setup-record.md).

## Invoking the Agent

The health advisor agent is invoked automatically when you ask a health-related question in Claude Code. You can
also target it directly using the `--agent` flag:

```bash
claude --agent over-50s-health:advisor
```

This sets the agent as the preferred delegate for the session. Claude Code routes health and wellness queries to it
automatically; for anything else, the default assistant responds.

Alternatively, just start Claude Code normally and ask a health question — the agent will activate based on your
query.

## Starting a Conversation

When you send your first message (even just "Hi"), the agent reads your context files and greets you with a brief
summary of your current health focus. If context files are missing, it will create them and prompt you to fill in
your initial information.

Once you've filled in your initial context files, you can begin interacting with the agent. Here are some ways to start:

**Initial Engagement:**

- "Where do you see the context files?" — the agent will analyze your context and provide a summary of your health
  profile, helping establish rapport.
- "Can you review my health information and suggest areas to focus on?"
- "What do you know about me so far?"

**Direct Queries:**

- "What strength training program would you recommend for me?"
- "Can you suggest a weekly meal plan that supports my goals?"
- "How can I improve my metabolic health based on my recent labs?"
- "What mobility exercises should I prioritize?"

**Specific Requests:**

- "Review my recent blood panel and explain what the trends mean"
- "Create a 4-week progressive workout plan for me"
- "Suggest supplements appropriate for my age and health status"

The agent will read your context files, provide evidence-based guidance with citations, and update your context files
as you share new information.

## Mobile and voice access

The agent can be used hands-free from a phone or tablet using two Claude Code features:

**Remote Control** — continues a local Claude Code session on any device:

1. Start a session on your Mac with the advisor agent active.
2. Run `/remote-control` to generate a session URL and QR code.
3. Scan the QR code in the Claude app or open the URL in a browser on your phone.
4. Your phone controls the local session with full access to your context files.

Your computer must stay on and the session must remain open. See the
[Claude Code Remote Control docs](https://code.claude.com/docs/en/remote-control) for details.

**Voice input** — speak your queries instead of typing:

Run `/voice` in your Claude Code session to enable voice input. Hold the spacebar to speak,
release to send. Transcription is free and does not count against rate limits. Responses are
text-only; voice output is not yet available.

Combined, `/remote-control` and `/voice` let you speak health queries from your phone while
the agent reads and updates your context files on your computer in the background.

> Available on Pro, Max, Team, and Enterprise plans.

## Safety and medical boundaries

- Educational guidance only, not diagnosis.
- Immediate referral for emergency symptoms.
- Always include a reminder to confirm with a healthcare professional.

## Evidence and citations

- Use credible, evidence-based sources.
- Provide citations with links in every response that includes recommendations.
- End responses with a **Sources** section.

## License

MIT License, Copyright (c) 2026 Alister Lewis-Bowen.
