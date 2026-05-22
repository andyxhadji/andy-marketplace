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

## Phase 2: Check Prerequisites

Check each prerequisite and record status.

### 2.1 Check Go

Run:
```bash
go version 2>/dev/null
```

**If exits 0 and output contains "go1.2[6-9]" or "go1.[3-9]":** Set `GO_INSTALLED=true`, extract version.
**Otherwise:** Set `GO_INSTALLED=false`.

### 2.2 Check glab

Run:
```bash
which glab >/dev/null 2>&1 && glab --version 2>/dev/null | head -1
```

**If exits 0:** Set `GLAB_INSTALLED=true`, extract version.
**Otherwise:** Set `GLAB_INSTALLED=false`.

### 2.3 Check sqlite3

Run:
```bash
which sqlite3 >/dev/null 2>&1 && sqlite3 --version 2>/dev/null | head -1
```

**If exits 0:** Set `SQLITE_INSTALLED=true`, extract version.
**Otherwise:** Set `SQLITE_INSTALLED=false`.

### 2.4 Check middleman

Run:
```bash
which middleman >/dev/null 2>&1 && middleman version 2>&1 | head -1
```

**If exits 0:** Set `MIDDLEMAN_INSTALLED=true`, extract version.
**Otherwise:** Set `MIDDLEMAN_INSTALLED=false`.

### 2.5 Check kata

Run:
```bash
which kata >/dev/null 2>&1 && kata version 2>&1 | head -1
```

**If exits 0:** Set `KATA_INSTALLED=true`, extract version.
**Otherwise:** Set `KATA_INSTALLED=false`.

### 2.6 Check state directory

Run:
```bash
test -d ~/.auto-andy && echo "exists" || echo "missing"
```

Set `STATE_DIR_EXISTS` accordingly.

### 2.7 Check GitLab token

Run:
```bash
test -n "$MIDDLEMAN_GITLAB_FLATIRON_TOKEN" && echo "set" || echo "missing"
```

Set `TOKEN_SET` accordingly.

### 2.8 Check middleman config

Run:
```bash
test -f ~/.config/middleman/config.toml && echo "exists" || echo "missing"
```

Set `MIDDLEMAN_CONFIG_EXISTS` accordingly.

### 2.9 Display Status Table

Display:
```
## Prerequisite Status

| Prerequisite | Status | Version/Details |
|--------------|--------|-----------------|
| Go 1.26+ | $GO_STATUS | $GO_VERSION |
| glab | $GLAB_STATUS | $GLAB_VERSION |
| sqlite3 | $SQLITE_STATUS | $SQLITE_VERSION |
| middleman | $MIDDLEMAN_STATUS | $MIDDLEMAN_VERSION |
| kata | $KATA_STATUS | $KATA_VERSION |
| State directory | $STATE_DIR_STATUS | ~/.auto-andy |
| GitLab token | $TOKEN_STATUS | MIDDLEMAN_GITLAB_FLATIRON_TOKEN |
| Middleman config | $CONFIG_STATUS | ~/.config/middleman/config.toml |
```

Where status is "OK" (green checkmark implied) or "Missing".

**If `CHECK_ONLY=true`:** Stop here and exit.
