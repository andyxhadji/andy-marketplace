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

## Phase 1: Detect Environment

### 1.1 Detect OS

Run:
```bash
uname -s
```

Store result as `OS`:
- `Darwin` → macOS
- `Linux` → Linux

**If neither:** Display error and exit:
```
Error: Unsupported operating system. auto-andy setup supports macOS and Linux only.
```

### 1.2 Detect Shell

Run:
```bash
basename "$SHELL"
```

Store result as `SHELL_NAME`:
- `zsh` → zsh
- `bash` → bash
- `fish` → fish

Determine `RC_FILE` based on shell:
- zsh: `~/.zshrc`
- bash: `~/.bashrc`
- fish: `~/.config/fish/config.fish`

**If unrecognized shell:** Default to `~/.profile` and warn:
```
Warning: Unrecognized shell '$SHELL_NAME'. Using ~/.profile for environment configuration.
```

### 1.3 Detect Package Manager

**For macOS:**
Run:
```bash
which brew >/dev/null 2>&1 && echo "brew" || echo "none"
```

**If "none":** Display error and exit:
```
Error: Homebrew is not installed.

Install Homebrew first:
  /bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

Then re-run: /auto-andy:setup
```

**For Linux:**
Run:
```bash
which apt >/dev/null 2>&1 && echo "apt" || (which dnf >/dev/null 2>&1 && echo "dnf" || echo "none")
```

Store result as `PKG_MANAGER` (apt, dnf, or none).

**If "none" on Linux:** Display error and exit:
```
Error: No supported package manager found (apt or dnf required).
```

### 1.4 Display Environment Summary

Display:
```
## Environment Detected

| Setting | Value |
|---------|-------|
| OS | $OS |
| Shell | $SHELL_NAME |
| RC File | $RC_FILE |
| Package Manager | $PKG_MANAGER |
```
