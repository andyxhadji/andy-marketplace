# auto-andy Plugin

Claude Code plugin for automated MR comment triage and response.

## Overview

Integrates middleman (MR sync), kata (task tracking), and roborev (code review) into a daily workflow:

```
middleman (MR comments) → triage → kata tasks → address --auto → approval doc → post-comments → MR responses
```

## Skills

### triage

Query middleman for new MR comments and create kata tasks.

```
/auto-andy:triage                  # Process all repos
/auto-andy:triage --repo ml-llms   # Single repo
/auto-andy:triage --dry-run        # Preview only
```

### address

Process kata tasks, auto-address high-confidence tasks, create specs for complex ones.

```
/auto-andy:address --auto              # Process all projects
/auto-andy:address --auto --project ml-llms  # Single project
/auto-andy:address --auto --dry-run    # Preview only
/auto-andy:address --auto --reset      # Clear pending approvals first
```

### post-comments

Post approved responses to MRs.

```
/auto-andy:post-comments           # Post all
/auto-andy:post-comments --dry-run # Preview only
```

## State Directory

Global state in `~/.auto-andy/`:

| File | Purpose |
|------|---------|
| `config.toml` | Optional configuration overrides |
| `last-triage.txt` | Timestamp of last triage run |
| `pending-approval.md` | Comments awaiting human review |
| `history/` | Archived approval docs |
| `specs/` | Spec handoffs for complex tasks |

## Prerequisites

- **middleman** running and syncing repos
- **kata** CLI available with projects matching repo names
- Environment variables for API access:
  - `MIDDLEMAN_GITLAB_FLATIRON_TOKEN`
  - `MIDDLEMAN_GITHUB_TOKEN` (if using GitHub)

## Daily Workflow

1. Run triage (can be automated/cron):
   ```
   /auto-andy:triage
   ```

2. When ready to address comments:
   ```
   /auto-andy:address --auto
   ```

3. Review `~/.auto-andy/pending-approval.md` and edit responses

4. Post approved comments:
   ```
   /auto-andy:post-comments
   ```

## Making Changes

**CRITICAL:** Always bump version in `.claude-plugin/plugin.json` before ANY change.

After changes:
1. Commit and push
2. Clear cache: `rm -rf ~/.claude/cache/* ~/.claude/plugins/cache/`
3. Restart Claude Code
