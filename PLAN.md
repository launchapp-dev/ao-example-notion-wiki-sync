# Notion Wiki Sync & Staleness Monitor — Build Plan

## Overview

A scheduled pipeline that audits a Notion wiki workspace for stale, broken, or outdated documentation. It scans all pages in a designated database, scores freshness, cross-references code mentions against the actual repo, flags problems, and generates actionable reports — all on a weekly cron.

## Architecture

### Agents

| Agent | Model | Role |
|---|---|---|
| **page-scanner** | claude-haiku-4-5 | Connects to Notion, enumerates all wiki pages, extracts metadata (last edited, author, links, code refs) |
| **staleness-analyzer** | claude-sonnet-4-6 | Scores each page for freshness, detects broken links, identifies outdated content patterns |
| **code-cross-referencer** | claude-sonnet-4-6 | Checks code references in wiki pages against the actual codebase — flags removed functions, renamed files, deleted features |
| **report-writer** | claude-sonnet-4-6 | Generates a comprehensive staleness report in Markdown, creates per-page action items |
| **wiki-updater** | claude-sonnet-4-6 | Posts comments on stale Notion pages with suggested fixes, exports critical docs to git-tracked markdown backup |
| **review-auditor** | claude-haiku-4-5 | Reviews the final report and actions for accuracy before committing |

### Phase Pipeline

```
scan-pages → analyze-staleness → cross-reference-code → write-report → review-audit → update-wiki
                                                                          ↑                |
                                                                          └── (rework) ────┘
```

1. **scan-pages** (agent: page-scanner)
   - Connect to Notion workspace via Notion MCP
   - Query the designated wiki database for all pages
   - Extract per-page: title, last_edited_time, last_edited_by, page content, inline links, code blocks/references
   - Write structured page inventory to `output/page-inventory.json`
   - Capabilities: reads Notion, writes files

2. **analyze-staleness** (agent: staleness-analyzer)
   - Read `output/page-inventory.json`
   - Score each page:
     - **current**: edited within 30 days, no issues
     - **stale**: not edited in 30-90 days
     - **critical**: not edited in 90+ days, or contains broken links
     - **archive-candidate**: not edited in 180+ days with no recent views
   - Detect broken internal links (pages that reference deleted Notion pages)
   - Write `output/staleness-scores.json` with per-page verdicts
   - Capabilities: reads files, writes files

3. **cross-reference-code** (agent: code-cross-referencer)
   - Read `output/page-inventory.json` for code references found in wiki pages
   - Use git MCP to check if referenced files, functions, classes, endpoints still exist in the repo
   - Flag: removed files, renamed functions, deprecated APIs, deleted features
   - Write `output/code-references.json` with match status per reference
   - Capabilities: reads files, reads git repo, writes files

4. **write-report** (agent: report-writer)
   - Read all output JSON files
   - Generate `output/staleness-report.md`:
     - Executive summary (total pages, % stale, % critical)
     - Per-page breakdown with status, issues found, recommended action
     - Code reference mismatches section
     - Priority action items (sorted by severity)
   - Generate `output/actions.json` with structured action items for the updater
   - Capabilities: reads files, writes files

5. **review-audit** (agent: review-auditor)
   - Review `output/staleness-report.md` and `output/actions.json`
   - Decision contract: { verdict: "advance" | "rework", reasoning: "..." }
   - Check: report is complete, actions are reasonable, no false positives in code refs
   - If issues found: rework → write-report
   - Capabilities: reads files

6. **update-wiki** (agent: wiki-updater)
   - Read `output/actions.json`
   - For pages marked stale/critical: post a Notion comment with staleness warning and suggested next steps
   - For pages marked archive-candidate: add an "Archive Candidate" tag/property
   - Export all critical-status pages to `backup/` as markdown files (git-tracked safety net)
   - Commit the backup exports and report to git
   - Capabilities: writes Notion comments, reads/writes files, git commit

### MCP Servers

| Server | Package | Used By | Purpose |
|---|---|---|---|
| **notion** | `@notionhq/notion-mcp-server` | page-scanner, wiki-updater | Read pages, post comments, update properties |
| **git** | `@modelcontextprotocol/server-git` | code-cross-referencer, wiki-updater | Check file existence, read repo state, commit |
| **filesystem** | `@modelcontextprotocol/server-filesystem` | all agents | Read/write JSON and Markdown output files |

### Workflow Routing

- **review-audit** has a decision contract:
  - `advance` → proceeds to update-wiki
  - `rework` → loops back to write-report (max 2 rework attempts)

### Schedule

- Weekly cron: `0 9 * * 1` (every Monday at 9 AM)
- Can also be triggered manually via `ao workflow run`

## File Structure

```
examples/notion-wiki-sync/
├── .ao/
│   └── workflows/
│       ├── agents.yaml
│       ├── phases.yaml
│       ├── workflows.yaml
│       ├── mcp-servers.yaml
│       └── schedules.yaml
├── config/
│   └── wiki-config.yaml          # Wiki database ID, repo path, staleness thresholds
├── output/                        # Generated during runs (gitignored except reports)
│   ├── page-inventory.json
│   ├── staleness-scores.json
│   ├── code-references.json
│   ├── staleness-report.md
│   └── actions.json
├── backup/                        # Git-tracked markdown exports of critical pages
├── templates/
│   └── report-template.md        # Template for the staleness report
├── CLAUDE.md                      # Project-level instructions
├── README.md                      # Setup guide, what this does, how to configure
└── .gitignore
```

## Configuration (config/wiki-config.yaml)

```yaml
notion:
  database_id: "your-wiki-database-id"    # The Notion database containing wiki pages
  workspace_name: "Your Workspace"

repository:
  path: "/path/to/your/codebase"          # Repo to cross-reference code mentions against

thresholds:
  stale_days: 30                           # Days without edit → stale
  critical_days: 90                        # Days without edit → critical
  archive_days: 180                        # Days without edit → archive candidate

actions:
  post_comments: true                      # Post staleness warnings as Notion comments
  tag_archive_candidates: true             # Add archive tag to old pages
  export_critical: true                    # Export critical pages to markdown backup
  auto_commit_backup: true                 # Git commit the backup exports
```

## Key Design Decisions

1. **Haiku for scanning, Sonnet for analysis** — scanning is high-volume mechanical work (list pages, extract fields); analysis requires judgment about what's stale vs. intentionally stable.

2. **Separate code cross-referencing phase** — keeps the staleness scoring (time-based) cleanly separated from code validation (repo-based). Either can be improved independently.

3. **Review gate before wiki updates** — posting comments to Notion is a visible, hard-to-undo action. The review phase catches false positives before they annoy page owners.

4. **Markdown backup as safety net** — even if Notion has issues, critical documentation is preserved in git. This also enables diff-based change detection across runs.

5. **Weekly schedule, not daily** — wiki staleness is a slow-moving problem. Weekly audits are frequent enough to catch rot without creating noise.
