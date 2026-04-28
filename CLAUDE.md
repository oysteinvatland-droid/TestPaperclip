# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Repo Is

**TestPaperclip** is a configuration/data repository for a [Paperclip](https://paperclip.sh) instance — a multi-agent orchestration platform. There is no source code to build or test. All behavior is defined through agent instruction files (`.md`) and JSON configuration.

The running instance hosts **TechBrief Daily**: a multi-agent pipeline where a CEO agent orchestrates a Researcher → Writer → Editor workflow to produce a daily tech newsletter.

## Instance Layout

All instance data lives under `.paperclip/instances/default/`:

- `config.json` — server, database, storage, secrets config
- `.env` — API credentials (not committed)
- `companies/{company-id}/agents/{agent-id}/instructions/` — per-agent instruction files
- `data/run-logs/` — NDJSON execution logs per agent run
- `data/backups/` — auto-generated PostgreSQL backups (60-min intervals, 30-day retention)
- `db/` — embedded PostgreSQL data directory (port 54329)
- `workspaces/{company-id}/` — shared filesystem where agents write outputs

## Server Configuration

From `config.json`:
- **API server**: `127.0.0.1:3100` (serves UI + REST API)
- **Database**: embedded PostgreSQL on port `54329`
- **Auth mode**: `local_trusted`
- **Storage**: local disk
- **Secrets**: local encrypted

## Agent Architecture

Agents are defined by four instruction files in their `instructions/` directory:

| File | Purpose |
|------|---------|
| `AGENTS.md` | Role definition, assigned tasks, and workflow logic |
| `SOUL.md` | Persona, values, and strategic posture |
| `HEARTBEAT.md` | Step-by-step execution checklist run on each wake cycle |
| `TOOLS.md` | Available tools declaration |

### Current Agents (company `8aa75994-...`)

| Agent ID (prefix) | Role |
|---|---|
| `3f622672` | CEO — orchestrates the full pipeline |
| `d518e2f9` | Researcher — finds top tech stories, writes `output/stories.md` |
| `282f8b0e` | Editor — scores newsletter, approves or rejects |
| `ffba375e` | CEO (backup/copy) |

## API Patterns

Agents interact with the Paperclip server via REST. All mutating requests require the `X-Paperclip-Run-Id` header. Auth is Bearer token via `$PAPERCLIP_API_KEY`.

Key endpoints used in agent instructions:

```
GET  /api/agents/me                          — agent identity + company ID
GET  /api/agents/me/inbox-lite               — assigned tasks
GET  /api/companies/{id}/issues?assigneeAgentId=...&status=todo,in_progress,blocked
POST /api/issues/{id}/checkout               — claim a task (409 if already claimed)
POST /api/issues/{id}/mark-done              — complete with optional comment
POST /api/companies/{id}/issues              — create a new task/subtask
```

## Newsletter Pipeline Flow

```
CEO wakes → creates "Find top 5 tech stories" → assigns to Researcher
Researcher wakes → checkouts task → searches web → writes output/stories.md → marks done
CEO wakes → creates "Draft newsletter" → assigns to Writer
Writer wakes → reads stories.md → writes output/newsletter.md → marks done
CEO wakes → creates "Evaluate and score" → assigns to Editor
Editor wakes → reads newsletter.md → scores (Clarity/Engagement/Accuracy/Structure)
  ≥ 8: approve → notify CEO
  < 8: reject with feedback → CEO reassigns Writer for revision
```

## Modifying Agent Behavior

To change what an agent does, edit its instruction files in:
```
.paperclip/instances/default/companies/8aa75994-9b39-4e50-a8a4-90fb100fe663/agents/{agent-id}/instructions/
```

Changes take effect on the agent's next heartbeat cycle — no restart required.
