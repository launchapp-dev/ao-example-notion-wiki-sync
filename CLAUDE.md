# Notion Wiki Sync — Agent Context

This project is a scheduled AO pipeline that audits a Notion wiki for staleness and keeps documentation healthy.

## What This Repo Does

Connects to a Notion workspace, scans all wiki pages in a designated database, scores each page for freshness, validates code references in those pages against the actual codebase, and takes action: posting comments on stale pages, tagging archive candidates, and exporting critical pages to git.

## Key Configuration

- **config/wiki-config.yaml** — All runtime settings: Notion database ID, repo path, staleness thresholds, which actions are enabled
- **NOTION_TOKEN** env var — Notion integration token (never hardcode this)
- **REPO_PATH** env var — overrides config/wiki-config.yaml repository.path if set

## Output Files

All intermediate files go to `output/`. These are transient — regenerated each run.
Files in `backup/` are git-tracked and meant to persist across runs.

The staleness report in `output/staleness-report.md` is the primary deliverable.
Historical reports are archived to `backup/reports/<date>-staleness-report.md`.

## Staleness Thresholds

Default (configurable in wiki-config.yaml):
- **current**: edited within 30 days — no action
- **stale**: 30–90 days — post a comment
- **critical**: 90+ days OR broken links — post urgent comment + export backup
- **archive-candidate**: 180+ days — tag as archive candidate

## Phase Dependencies

Phases must run in order — each reads the previous phase's output:
1. scan-pages → writes output/page-inventory.json
2. analyze-staleness → reads page-inventory.json, writes staleness-scores.json
3. cross-reference-code → reads page-inventory.json, writes code-references.json
4. write-report → reads all three, writes staleness-report.md + actions.json
5. review-audit → reads report + actions, issues advance/rework verdict
6. update-wiki → reads actions.json, posts to Notion, commits backup

## MCP Servers

- **notion** (`@notionhq/notion-mcp-server`): Read pages, post comments, update properties
- **git** (`@modelcontextprotocol/server-git`): Check file existence in repo, commit backup files
- **filesystem** (`@modelcontextprotocol/server-filesystem`): Read/write all local JSON and Markdown files

## Design Decisions

- **Haiku for scanning, Sonnet for analysis**: Scanning is mechanical and high-volume; analysis requires judgment
- **Review gate before Notion updates**: Posting comments is visible and hard to undo — false positives annoy page owners
- **Weekly schedule**: Wiki rot is slow-moving; weekly is frequent enough without creating noise
- **Separate code cross-referencing**: Keeps time-based staleness scoring independent from code validation — each can evolve independently
- **Markdown backup in git**: Even if Notion has issues, critical docs are preserved and diffable
