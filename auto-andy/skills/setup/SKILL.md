---
name: setup
description: Install and configure all prerequisites for auto-andy (middleman, kata, glab, tokens)
allowed-tools: [Bash, Read, Write, AskUserQuestion]
---

# Setup Auto-Andy Prerequisites

## Objective

Automatically install and configure all prerequisites needed to run the auto-andy plugin skills (triage, address, post-comments).

## Arguments

The user's argument (if any) is: $ARGUMENTS

**Supported flags:**
- `--check` - Check prerequisites only, don't install anything
- `--force` - Reinstall tools even if already installed
- `--skip-token` - Skip token configuration prompt

## Prerequisites

| Tool | Purpose |
|------|---------|
| Go 1.26+ | Required to install middleman and kata |
| glab | GitLab CLI for posting comments |
| sqlite3 | Query middleman database |
| middleman | MR sync daemon |
| kata | Task tracking CLI |

## Phase 0: Parse Arguments

Parse `$ARGUMENTS` for:
- `--check` - Set `CHECK_ONLY=true`
- `--force` - Set `FORCE_INSTALL=true`
- `--skip-token` - Set `SKIP_TOKEN=true`
- Default: `CHECK_ONLY=false`, `FORCE_INSTALL=false`, `SKIP_TOKEN=false`
