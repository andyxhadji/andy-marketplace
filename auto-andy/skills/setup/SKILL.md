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

## Phase 3: Install Missing Prerequisites

Install each missing prerequisite in dependency order. Skip if already installed (unless `FORCE_INSTALL=true`).

### 3.1 Install Go (if missing)

**If `GO_INSTALLED=false` or `FORCE_INSTALL=true`:**

**macOS:**
```bash
brew install go
```

**Linux (apt):**
```bash
sudo apt update && sudo apt install -y golang
```

**Linux (dnf):**
```bash
sudo dnf install -y golang
```

**Verify installation:**
```bash
go version
```

**If fails:** Display error:
```
Error: Failed to install Go.

Manual installation:
  macOS: brew install go
  Linux: sudo apt install golang
  Or visit: https://go.dev/doc/install
```

### 3.2 Install glab (if missing)

**If `GLAB_INSTALLED=false` or `FORCE_INSTALL=true`:**

**macOS:**
```bash
brew install glab
```

**Linux (apt):**
```bash
sudo apt install -y glab
```

**Linux (dnf):**
```bash
sudo dnf install -y glab
```

**Verify installation:**
```bash
glab --version
```

**If fails:** Display error:
```
Error: Failed to install glab.

Manual installation:
  macOS: brew install glab
  Linux: See https://gitlab.com/gitlab-org/cli/-/releases
```

### 3.3 Install middleman (if missing)

**If `MIDDLEMAN_INSTALLED=false` or `FORCE_INSTALL=true`:**

Run:
```bash
go install go.kenn.io/middleman@latest
```

**If fails:** Retry once:
```bash
go install go.kenn.io/middleman@latest
```

**If still fails:** Display error:
```
Error: Failed to install middleman.

Manual installation:
  go install go.kenn.io/middleman@latest

Ensure ~/go/bin is in your PATH.
```

**Verify installation:**
```bash
which middleman && middleman version
```

### 3.4 Install kata (if missing)

**If `KATA_INSTALLED=false` or `FORCE_INSTALL=true`:**

Run:
```bash
go install go.kenn.io/kata/cmd/kata@latest
```

**If fails:** Retry once:
```bash
go install go.kenn.io/kata/cmd/kata@latest
```

**If still fails:** Display error:
```
Error: Failed to install kata.

Manual installation:
  go install go.kenn.io/kata/cmd/kata@latest

Ensure ~/go/bin is in your PATH.
```

**Verify installation:**
```bash
which kata && kata version
```

### 3.5 Create state directory (if missing)

**If `STATE_DIR_EXISTS=false`:**

Run:
```bash
mkdir -p ~/.auto-andy/pending ~/.auto-andy/specs ~/.auto-andy/history
```

**Verify:**
```bash
ls -la ~/.auto-andy/
```

## Phase 4: Configure

### 4.1 Configure GitLab Token

**If `TOKEN_SET=false` and `SKIP_TOKEN=false`:**

Prompt user using AskUserQuestion:
```
Enter your GitLab personal access token for git.the.flatiron.com (needs api scope):

To create a token:
1. Go to https://git.the.flatiron.com/-/user_settings/personal_access_tokens
2. Create a token with 'api' scope
3. Copy and paste the token here
```

Store response as `GITLAB_TOKEN`.

**If user cancels or provides empty token:**
Display warning:
```
Warning: No token provided. You can set it later by adding to your shell RC file:
  export MIDDLEMAN_GITLAB_FLATIRON_TOKEN="your-token-here"
```
Set `TOKEN_CONFIGURED=false` and continue.

**If token provided:**

Add to shell RC file based on `SHELL_NAME`:

**zsh/bash:** Append to `$RC_FILE`:
```bash
echo '' >> $RC_FILE
echo '# Auto-andy GitLab token' >> $RC_FILE
echo 'export MIDDLEMAN_GITLAB_FLATIRON_TOKEN="$GITLAB_TOKEN"' >> $RC_FILE
```

**fish:** Append to `~/.config/fish/config.fish`:
```bash
mkdir -p ~/.config/fish
echo '' >> ~/.config/fish/config.fish
echo '# Auto-andy GitLab token' >> ~/.config/fish/config.fish
echo 'set -gx MIDDLEMAN_GITLAB_FLATIRON_TOKEN "$GITLAB_TOKEN"' >> ~/.config/fish/config.fish
```

**Export for current session:**
```bash
export MIDDLEMAN_GITLAB_FLATIRON_TOKEN="$GITLAB_TOKEN"
```

Set `TOKEN_CONFIGURED=true`.

### 4.2 Configure Middleman

**If `MIDDLEMAN_CONFIG_EXISTS=false`:**

Create directory:
```bash
mkdir -p ~/.config/middleman
```

Write config file using Write tool to `~/.config/middleman/config.toml`:

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

Set `CONFIG_CREATED=true`.

**If config already exists:**
Display:
```
Middleman config already exists at ~/.config/middleman/config.toml - skipping.
```
Set `CONFIG_CREATED=false`.

## Phase 5: Verify Installation

Run smoke tests to confirm everything works.

### 5.1 Verify tools

Run each verification and record results:

```bash
# middleman
middleman --help >/dev/null 2>&1 && echo "middleman: OK" || echo "middleman: FAIL"

# kata
kata --help >/dev/null 2>&1 && echo "kata: OK" || echo "kata: FAIL"

# glab
glab --version >/dev/null 2>&1 && echo "glab: OK" || echo "glab: FAIL"

# sqlite3
sqlite3 --version >/dev/null 2>&1 && echo "sqlite3: OK" || echo "sqlite3: FAIL"
```

### 5.2 Verify token

```bash
test -n "$MIDDLEMAN_GITLAB_FLATIRON_TOKEN" && echo "GitLab token: OK" || echo "GitLab token: FAIL"
```

### 5.3 Verify state directory

```bash
test -d ~/.auto-andy && echo "State directory: OK" || echo "State directory: FAIL"
```

### 5.4 Verify middleman config

```bash
test -f ~/.config/middleman/config.toml && echo "Middleman config: OK" || echo "Middleman config: FAIL"
```

### 5.5 Display Verification Results

Display:
```
## Verification Results

| Component | Status |
|-----------|--------|
| middleman | $MIDDLEMAN_VERIFY |
| kata | $KATA_VERIFY |
| glab | $GLAB_VERIFY |
| sqlite3 | $SQLITE_VERIFY |
| GitLab token | $TOKEN_VERIFY |
| State directory | $STATE_VERIFY |
| Middleman config | $CONFIG_VERIFY |
```

**If any FAIL:** Display warning:
```
Warning: Some components failed verification. See details above.
```
