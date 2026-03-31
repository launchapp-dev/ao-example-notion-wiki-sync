# Notion Wiki Sync & Staleness Monitor

Automatically audit your Notion wiki for stale, broken, and outdated documentation — on a weekly schedule. Scans every page, scores freshness, validates code references against your actual repo, generates a report, and posts comments directly on stale pages.

## What It Does

Every Monday at 9 AM, this pipeline:
1. **Scans** all pages in your Notion wiki database
2. **Scores** each page: current → stale → critical → archive-candidate
3. **Cross-references** code mentions (file paths, function names, API routes) against your real codebase
4. **Generates** a structured staleness report with executive summary and per-page breakdown
5. **Reviews** the report automatically for false positives before taking any action
6. **Updates** Notion: posts comments on stale pages, tags archive candidates, exports critical docs to git

## Workflow Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                   WEEKLY CRON (Mon 9 AM)                        │
└───────────────────────────┬─────────────────────────────────────┘
                            │
                            ▼
                    ┌───────────────┐
                    │  scan-pages   │  page-scanner (Haiku)
                    │  Notion MCP   │  → output/page-inventory.json
                    └───────┬───────┘
                            │
                            ▼
                 ┌─────────────────────┐
                 │  analyze-staleness  │  staleness-analyzer (Sonnet)
                 │  Score & classify   │  → output/staleness-scores.json
                 └──────────┬──────────┘
                            │
                            ▼
              ┌──────────────────────────┐
              │  cross-reference-code    │  code-cross-referencer (Sonnet)
              │  Filesystem repo check   │  → output/code-references.json
              └────────────┬─────────────┘
                           │
                           ▼
                  ┌─────────────────┐
                  │  write-report   │◄──────────────────┐
                  │  Synthesize all │                   │ rework
                  └────────┬────────┘                   │ (max 2x)
                           │                            │
                           ▼                            │
                  ┌─────────────────┐                   │
                  │  review-audit   │──── rework ───────┘
                  │  QA gate        │
                  └────────┬────────┘
                           │ advance
                           ▼
                  ┌─────────────────┐
                  │  update-wiki    │  wiki-updater (Sonnet)
                  │  Post comments  │  → Notion comments
                  │  Tag archives   │  → backup/ exports
                  │  Export + commit│  → git commit
                  └─────────────────┘
```

## Quick Start

### 1. Prerequisites

- [AO CLI](https://github.com/launchapp-dev/ao) installed
- A Notion integration token with access to your wiki database
- Node.js 18+ (for MCP servers via npx)

### 2. Configure

```bash
cd examples/notion-wiki-sync

# Edit config/wiki-config.yaml
# Set your Notion database ID, repo path, and thresholds
```

**Get your Notion database ID:**
Open the database in Notion → copy the URL → the long hex string is the database ID.

**Create a Notion integration:**
Go to https://www.notion.so/my-integrations → New integration → copy the token.
Share your wiki database with the integration (database settings → Connections).

### 3. Set Environment Variables

```bash
export NOTION_TOKEN="secret_xxxxxxxxxxxxxxxxxxxx"
export REPO_PATH="/path/to/your/codebase"
```

### 4. Run

```bash
# Start the daemon (picks up the weekly schedule automatically)
ao daemon start

# Or run immediately
ao workflow run wiki-staleness-audit

# Dry run (no Notion updates)
ao workflow run scan-only
```

### 5. Watch It Work

```bash
ao daemon stream --pretty
```

## Agents

| Agent | Model | Role |
|---|---|---|
| **page-scanner** | claude-haiku-4-5 | Enumerates all wiki pages and extracts metadata, links, and code references |
| **staleness-analyzer** | claude-sonnet-4-6 | Scores pages current/stale/critical/archive-candidate, detects broken links |
| **code-cross-referencer** | claude-sonnet-4-6 | Validates code mentions against the actual repository using filesystem MCP |
| **report-writer** | claude-sonnet-4-6 | Generates the full staleness report and structured action plan |
| **review-auditor** | claude-haiku-4-5 | Quality gate — checks for false positives before any Notion updates |
| **wiki-updater** | claude-sonnet-4-6 | Posts Notion comments, tags archive candidates, exports + commits backups |

## AO Features Demonstrated

- **Scheduled workflows** — weekly cron trigger (`0 9 * * 1`)
- **Multi-agent pipeline** — 6 specialized agents with clear handoffs
- **Decision contracts** — review-audit phase with `advance`/`rework` verdicts
- **Rework loops** — report-writer retried up to 2x if review finds issues
- **Multiple MCP servers** — Notion MCP and Filesystem MCP working together
- **Mixed agent models** — Haiku for high-volume scanning, Sonnet for reasoning
- **Manual-safe actions** — review gate before any visible Notion changes
- **Git-tracked backup** — critical pages exported and committed automatically

## Workflows

| Workflow | Description |
|---|---|
| `wiki-staleness-audit` | Full weekly audit (default) |
| `scan-only` | Scan and report without posting to Notion — safe for dry runs |
| `report-refresh` | Re-generate report from existing scan data |

## Requirements

| Requirement | Details |
|---|---|
| `NOTION_TOKEN` | Notion integration token (secret_xxx) |
| `REPO_PATH` | Absolute path to codebase for code cross-referencing |
| Notion database | A database of wiki pages shared with your integration |
| Node.js 18+ | For npx MCP servers |

## Output Files

| File | Description |
|---|---|
| `output/page-inventory.json` | All scanned pages with metadata (transient) |
| `output/staleness-scores.json` | Per-page staleness verdicts (transient) |
| `output/code-references.json` | Code reference validation results (transient) |
| `output/staleness-report.md` | Full report (kept in git) |
| `output/actions.json` | Structured action plan (transient) |
| `backup/<page>.md` | Exported critical pages (git-tracked) |
| `backup/reports/<date>-staleness-report.md` | Historical reports (git-tracked) |
