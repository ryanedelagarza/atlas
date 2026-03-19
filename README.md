# Atlas — Personal Chief of Staff System

A knowledge management and portfolio intelligence system built on top of Claude Code, Azure OpenAI, and plain Markdown files.

---

## What it is

Atlas is a Chief of Staff that lives in a folder. It knows what I'm working on, tracks health across my project portfolio, processes information I drop into it, and generates briefings without me asking. The design goal is to reduce the cognitive load of staying on top of 12+ active projects while making sure the right things surface at the right time.

The name comes from carrying weight — not from the app.

---

## Why I built it

I manage a portfolio of content and platform projects at an education company. The problem I kept running into is what I call the "January problem": I'd come back from a break and have no idea what state anything was in, who was blocked, or what had quietly slipped. The fix shouldn't be a dashboard I have to manually update — it should be a system that knows.

---

## Design philosophy

**Plain text as the foundation.** Everything lives in Markdown files in a structured directory. No proprietary database, no vendor lock-in. If every AI tool disappeared tomorrow, the knowledge base is still readable, searchable, and useful.

**PARA for structure.** The folder organization follows Tiago Forte's PARA method: Projects (active work with a deadline), Areas (ongoing responsibilities), Resources (reference material), Archive. This gives a natural home for everything without having to decide.

**Chief of Staff as the interaction model.** You talk to one agent — Atlas — and it routes internally to specialists. You never see the seams. Atlas has a team: Scout (project health), Scribe (ingestion), Herald (briefings), Archivist (knowledge retrieval), Sentinel (watchdog), Curator (self-improvement). Each is a Markdown file with a persona, trigger conditions, and instructions. Routing happens by reading frontmatter YAML, not by loading every agent fully.

**Automation is acceleration, not dependency.** The system is designed to work manually first. Automation makes it faster. If Task Scheduler fails or Azure is down, I can open a Claude Code session and it all still works — I just have to ask instead of it running itself.

**Graceful degradation over fragile pipelines.** Every automated task logs to `_RECOVERY/automation-log.md`. Failures don't cascade. Data is never lost because it's on disk, not in session memory.

---

## Architecture

```
Atlas/
├── _AGENT/             # Agent team: CLAUDE.md router + 7 agent SKILL.md files
├── _INBOX/             # Drop zone: raw/ (unprocessed), processed/, filed/
├── _BRIEFINGS/         # Daily briefings + HTML dashboard
├── _CURATOR/           # Decision log + calibration queue
├── _RECOVERY/          # Automation log, health check, changelog
├── _SCRIPTS/           # Python automation engine
│   ├── automation-runner.py
│   ├── run-task.bat
│   └── tasks/          # 9 task modules
├── lib/                # Shared model invocation layer
│   ├── model_invoker.py
│   └── model_registry.py
├── 1_PROJECTS/         # One folder per active project (PARA)
├── 2_AREAS/            # Ongoing responsibilities
├── 3_RESOURCES/        # Templates, reference, team docs
└── 4_ARCHIVE/          # Completed or inactive projects
```

### Two interaction modes

**Conversational (Claude Code session):** Open the Atlas folder in Claude Code. CLAUDE.md loads, Atlas reads its startup sequence (project registry, watchlist rules, system map), scans the inbox for unprocessed items, and greets you with what matters. From there it's a conversation — drop in a meeting transcript, ask for a briefing, query a project.

**Automated (Windows Task Scheduler + Azure OpenAI):** Nine Python task modules run on schedule via `_SCRIPTS/automation-runner.py`. They read the knowledge base, call Azure OpenAI, write results back to Markdown files, and log everything. No Claude Code session needed.

### Automation schedule

| Task | Model | When | What it does |
|------|-------|------|-------------|
| `smartsheet_scan` | gpt-4o | Daily 6:45 AM | Pull project progress % and health from Smartsheet |
| `jira_scan` | gpt-4o | Daily 7:00 AM | Scan all 12 projects in Jira by component, update STATUS.md files |
| `confluence_scan` | gpt-4o + gpt-4o-mini | Tue/Thu 7:15 AM | Monitor 5 Confluence spaces, filter for relevance, extract to digest |
| `morning_briefing` | gpt-5-chat | Daily 7:30 AM | 6-section briefing: attention items, actions, deadlines, portfolio health, decisions, intel |
| `dashboard_refresh` | gpt-4o-mini | Daily 8:00 AM | JSON data refresh |
| `html_dashboard` | gpt-4o | Daily 8:05 AM | Render HTML dashboard from briefing data |
| `inbox_scan` | gpt-4o | Every 4h | File inbox items, log confidence scores to Curator |
| `sentinel_watchdog` | gpt-5-mini | Every 4h | Portfolio risk scan — staleness, SPOF, resource concentration |
| `cross_reference` | gpt-4o | Weekly Mon | Person-project matrix, dependency map, risk flags |

### Data flow

```
Smartsheet (6:45) --> STATUS.md files
Jira (7:00)       --> STATUS.md files
Confluence (7:15) --> confluence-digest.md
All three         --> morning_briefing (7:30) --> _BRIEFINGS/daily/YYYY-MM-DD.md
Briefing          --> html_dashboard (8:05)   --> _BRIEFINGS/dashboard.html
```

Every morning, the briefing reflects the current state of all 12 projects. Ryan wakes up to a current dashboard without doing anything.

### Model selection rationale

Routine extraction (JSON parsing, date stamping) uses `gpt-4o-mini`. Classification, health assessment, and structured analysis use `gpt-4o`. The morning briefing — the hardest task, requiring judgment calls across 12 projects — uses `gpt-5-chat`. Sentinel (portfolio-wide rule application) uses `gpt-5-mini`. This routing is hardcoded in each task module, not configurable, because it's not a decision that needs to change often.

### Atlassian integration

Jira and Confluence are connected via an internal HTTP proxy that bridges to `weldnorthed.atlassian.net`. In Claude Code sessions, this works via MCP (4 tools). In Python automation, the tasks call the proxy directly over HTTP with JSON-RPC 2.0 payloads and parse SSE responses. Both paths write to the same Markdown files.

---

## The agent team

Each agent is a SKILL.md file with YAML frontmatter (name, description, triggers) and a body (persona, instructions, references). The router (CLAUDE.md) loads only frontmatter at startup — 7 short YAML blocks instead of 7 full documents. When Ryan's intent matches a trigger, only that agent's full body is loaded.

The agents never reference each other in conversation. Ryan talks to Atlas. Atlas synthesizes. The team is invisible.

| Agent | Role |
|-------|------|
| Atlas (Chief of Staff) | Primary persona, routing, synthesis |
| Scout | Project health and risk analysis |
| Scribe | Parses raw input into structured knowledge |
| Herald | Briefings, dashboards, reports |
| Archivist | Knowledge retrieval and storage |
| Sentinel | Threshold monitoring and failure detection |
| Curator | Self-evaluation, filing confidence, calibration |

---

## The Curator feedback loop

Every inbox filing logs to `_CURATOR/decision-log.md` with a confidence score, the reasoning, and which project it was filed to. Filings under 70% confidence auto-queue to `_CURATOR/calibration-queue.md` for review. Filings under 50% are skipped entirely.

The idea is that over time, I review the low-confidence queue, correct misfilings, and the system learns what I mean. This isn't active ML — it's a structured log + calibration prompt pattern. The Curator reads historical decisions before making new ones.

---

## What's working

- Zero-touch morning briefings with current Jira data
- Portfolio health across 12 projects updated daily without manual exports
- HTML dashboard I can bookmark and refresh
- Inbox processing every 4 hours — I drop a meeting transcript and it gets filed
- Cross-reference analysis finding resource bottlenecks I didn't notice (e.g., one person named as lead on 7 projects)
- Sentinel catching staleness and surfacing it in briefings

---

## What's missing / deferred

**Slack integration.** No enterprise API access. The workaround is pasting Slack threads as files into the inbox — it works but isn't automatic.

**Calendar integration.** No API available. Atlas doesn't know what meetings happened or are coming. This means it can't flag "you had a meeting with X project's team and no notes were filed" — a gap I feel.

**Overnight processing (Bedrock).** Azure OpenAI runs scheduled tasks fine during the day. The plan was to add AWS Bedrock for off-hours batch processing, but Azure's persistent keys already provide 24/7 operation, so this became lower priority.

**Mature calibration.** The Curator feedback loop is instrumented and logging, but it needs more history before check-ins become meaningfully informative. It's accumulating.

**System map.** There's a stub for a code project catalog — a discovery scan that maps my dev environment and links code repos to project folders. Never actually run it. Atlas knows about projects from Smartsheet/Jira but not where the code lives.

**Slack/email as structured input.** The ingestion pipeline handles Markdown, plain text, and voice transcripts well. Binary files (.docx) are a known gap — they get logged as unreadable and I re-export them.

---

## What I'd do differently

**Start with the automation engine.** I designed the system assuming `claude -p` (Claude Code in pipe mode) would be the execution layer for scheduled tasks. That approach has a credential rotation problem — Claude Code work credentials expire every 5.5 hours. Switching to Azure OpenAI + Python task modules was the right call and should have been the starting assumption.

**The PARA structure is slightly wrong for a work portfolio.** PARA works well for personal knowledge management. For a director managing cross-functional projects, the distinction between "Projects" and "Areas" blurs — many of my "Areas" (like team health, budget oversight) have project-like urgency but no defined end. I've made it work, but a work-specific structure might be cleaner.

**Earlier Confluence monitoring.** I thought Jira was where decisions lived. Confluence is actually where the institutional context lives — spec changes, team charters, architectural decisions. Starting the Confluence scan earlier would have given Atlas better context from the start.

---

## Build timeline

All 4 sprints completed in approximately 4 days (2026-03-15 through 2026-03-18), faster than the planned 4-6 weeks. The acceleration came from having live Jira/Confluence API access early (planned for Sprint 4, got it in Sprint 2) and the Azure OpenAI automation engine being simpler and more reliable than the `claude -p` approach it replaced.

Sprint 1 (Day 1): Directory structure, agent team, templates, project registry populated from Smartsheet
Sprint 2 (Day 2-3): First briefing, live Jira/Confluence MCP, Smartsheet API, inbox processing
Sprint 3 (Day 3-4): Automation engine, HTML dashboard, Curator feedback loop, cross-referencing
Sprint 4 (Day 4): Jira automation, Confluence monitoring, full morning sequence operational

---

## Tech stack

- **Claude Code** — conversational interface, MCP integrations, agent routing
- **Azure OpenAI** — automation engine (gpt-4o, gpt-4o-mini, gpt-5-chat, gpt-5-mini)
- **Python** — task modules, model invocation layer, proxy calls
- **Windows Task Scheduler** — job scheduling
- **Atlassian (Jira + Confluence)** — source data via internal HTTP proxy
- **Smartsheet** — project progress and health source of truth
- **Markdown** — everything else

---

## Inspired by

This system was shaped significantly by conversations with a friend whose own personal OS demonstrated that a well-designed information environment could meaningfully change how a person works. The core insight — that the bottleneck isn't access to information, it's having the right information surfaced at the right time — came from watching his approach and adapting it to a different context.
