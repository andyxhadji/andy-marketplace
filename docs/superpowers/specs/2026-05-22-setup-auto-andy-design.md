# Setup Auto-Andy Skill Design

**Date:** 2026-05-22
**Status:** Approved

## Overview

A setup skill (`/auto-andy:setup`) that automatically installs and configures all prerequisites needed to run the auto-andy plugin skills (triage, address, post-comments).

## Prerequisites

| Tool | Purpose | Installation Method |
|------|---------|---------------------|
| Go 1.26+ | Required for middleman/kata install | `brew install go` (macOS) / `apt install golang` (Linux) |
| glab | GitLab CLI for API calls | `brew install glab` (macOS) / `apt install glab` (Linux) |
| sqlite3 | Query middleman database | Pre-installed on macOS/Linux |
| middleman | MR sync daemon | `go install go.kenn.io/middleman@latest` |
| kata | Task tracking CLI | `go install go.kenn.io/kata/cmd/kata@latest` |

## Environment Variables

| Variable | Purpose | Required |
|----------|---------|----------|
| `MIDDLEMAN_GITLAB_FLATIRON_TOKEN` | GitLab API access for Flatiron | Yes |
| `MIDDLEMAN_GITHUB_TOKEN` | GitHub API access | Optional |

## Directory Structure

```
~/.auto-andy/
├── pending/      # Per-MR approval documents awaiting review
├── specs/        # Spec handoffs for complex tasks
└── history/      # Archived approval documents

~/.config/middleman/
├── config.toml   # Middleman configuration
└── middleman.db  # SQLite database (created by middleman)
```

## Skill Phases

### Phase 1: Detect Environment

Detect the user's environment to determine appropriate installation commands.

**Detections:**
- OS: `uname -s` → darwin/Linux
- Shell: `basename "$SHELL"` → zsh/bash/fish
- Package manager: Check for brew (macOS), apt/dnf (Linux)
- Shell RC file:
  - zsh: `~/.zshrc`
  - bash: `~/.bashrc`
  - fish: `~/.config/fish/config.fish`

### Phase 2: Check Prerequisites

Check each prerequisite and record status.

**Checks:**
```bash
# Go
go version 2>/dev/null | grep -q "go1.2[6-9]\|go1.[3-9]"

# glab
which glab >/dev/null 2>&1

# sqlite3
which sqlite3 >/dev/null 2>&1

# middleman
which middleman >/dev/null 2>&1

# kata
which kata >/dev/null 2>&1

# State directory
test -d ~/.auto-andy

# Token
test -n "$MIDDLEMAN_GITLAB_FLATIRON_TOKEN"
```

**Output:** Status table showing installed/missing for each prerequisite.

### Phase 3: Install Missing Prerequisites

Install each missing prerequisite in dependency order.

**Order:**
1. Package manager tools (Go, glab) - these are independent
2. Go-based tools (middleman, kata) - depend on Go
3. State directory - independent

**Installation Commands:**

```bash
# macOS (brew)
brew install go
brew install glab

# Linux (apt)
sudo apt update && sudo apt install -y golang glab

# Linux (dnf)
sudo dnf install -y golang glab

# Go tools (all platforms)
go install go.kenn.io/middleman@latest
go install go.kenn.io/kata/cmd/kata@latest

# State directory
mkdir -p ~/.auto-andy/{pending,specs,history}
```

**Error Handling:**
- If brew not found on macOS: Display error with Homebrew install instructions
- If Go install fails: Display manual install instructions
- If go install fails: Retry once, then display manual command

### Phase 4: Configure

#### 4.1 Token Configuration

If `MIDDLEMAN_GITLAB_FLATIRON_TOKEN` is not set:

1. Prompt user: "Enter your GitLab personal access token for git.the.flatiron.com (needs api scope):"
2. Add export to shell RC file:

**zsh/bash:**
```bash
echo 'export MIDDLEMAN_GITLAB_FLATIRON_TOKEN="<token>"' >> ~/.zshrc
```

**fish:**
```bash
echo 'set -gx MIDDLEMAN_GITLAB_FLATIRON_TOKEN "<token>"' >> ~/.config/fish/config.fish
```

3. Export in current session for immediate use

#### 4.2 Middleman Config

If `~/.config/middleman/config.toml` does not exist, create it:

```toml
sync_interval = "5m"
github_token_env = "MIDDLEMAN_GITHUB_TOKEN"
default_platform_host = "github.com"
host = "127.0.0.1"
port = 8091

[activity]
view_mode = "threaded"
time_range = "7d"

[[platforms]]
type = "gitlab"
host = "git.the.flatiron.com"
token_env = "MIDDLEMAN_GITLAB_FLATIRON_TOKEN"

# Add repos via the Settings UI at http://localhost:8091
# Or add [[repos]] sections here:
#
# [[repos]]
# platform = "gitlab"
# platform_host = "git.the.flatiron.com"
# owner = "your-group"
# name = "your-repo"
```

### Phase 5: Verify

Run smoke tests to confirm everything works:

```bash
# Check tools are accessible
middleman --help >/dev/null 2>&1 && echo "middleman: OK" || echo "middleman: FAIL"
kata --help >/dev/null 2>&1 && echo "kata: OK" || echo "kata: FAIL"
glab --version >/dev/null 2>&1 && echo "glab: OK" || echo "glab: FAIL"
sqlite3 --version >/dev/null 2>&1 && echo "sqlite3: OK" || echo "sqlite3: FAIL"

# Check token is set
test -n "$MIDDLEMAN_GITLAB_FLATIRON_TOKEN" && echo "GitLab token: OK" || echo "GitLab token: FAIL"

# Check state directory
test -d ~/.auto-andy && echo "State directory: OK" || echo "State directory: FAIL"
```

### Phase 6: Report Summary

Display final summary:

```
## Auto-Andy Setup Complete

### Installed
- Go 1.26.3
- glab 1.52.0
- middleman (go.kenn.io/middleman@latest)
- kata (go.kenn.io/kata/cmd/kata@latest)

### Configured
- GitLab token added to ~/.zshrc
- Middleman config created at ~/.config/middleman/config.toml
- State directory created at ~/.auto-andy/

### Next Steps

1. Restart your shell or run: source ~/.zshrc
2. Start middleman: middleman
3. Add repos via http://localhost:8091 Settings
4. Run: /auto-andy:triage
```

## Arguments

| Flag | Description |
|------|-------------|
| `--check` | Check prerequisites only, don't install |
| `--force` | Reinstall even if already installed |
| `--skip-token` | Skip token configuration |

## Constraints

**Tool Restrictions:**
- **Bash:** Package manager commands, go install, mkdir, echo
- **Read:** Shell RC files, existing config files
- **Write:** Shell RC files, middleman config.toml
- **AskUserQuestion:** Token prompt only

**Security:**
- Token is prompted interactively, never logged
- Token is written only to user's RC file with appropriate permissions
- No tokens are hardcoded or committed

## Error States

| Error | Handling |
|-------|----------|
| No Homebrew on macOS | Display install instructions, exit |
| No sudo access on Linux | Display error, suggest running with sudo |
| Go install fails | Retry once, then display manual instructions |
| Token prompt cancelled | Warn but continue, token can be set later |
| Middleman config exists | Skip config creation, preserve existing |
