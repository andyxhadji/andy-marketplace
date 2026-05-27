# Setup Auto-Andy Skill Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Create `/auto-andy:setup` skill that auto-installs and configures all prerequisites for the auto-andy plugin.

**Architecture:** Single SKILL.md file following the existing auto-andy skill pattern. The skill executes phases sequentially: detect environment, check prerequisites, install missing tools, configure tokens and middleman, verify installation, and report summary.

**Tech Stack:** Bash for system commands, shell detection, package managers (brew/apt/dnf), Go toolchain for middleman/kata installation.

---

## File Structure

| File | Purpose |
|------|---------|
| `auto-andy/skills/setup/SKILL.md` | The setup skill definition |

This is a single-file implementation. The skill is entirely self-contained as a markdown document with embedded instructions for the agent to follow.

---

### Task 1: Create Setup Skill Directory and Frontmatter

**Files:**
- Create: `auto-andy/skills/setup/SKILL.md`

- [ ] **Step 1: Create the skill directory**

Run:
```bash
mkdir -p /Users/andy/.superset/worktrees/b63b1c22-154d-4a16-8535-4a258cf38d89/animated-light/auto-andy/skills/setup
```

- [ ] **Step 2: Create SKILL.md with frontmatter and overview**

Write to `auto-andy/skills/setup/SKILL.md`:

```markdown
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
```

- [ ] **Step 3: Verify file was created**

Run:
```bash
head -20 /Users/andy/.superset/worktrees/b63b1c22-154d-4a16-8535-4a258cf38d89/animated-light/auto-andy/skills/setup/SKILL.md
```

Expected: Shows frontmatter and overview section.

- [ ] **Step 4: Commit**

```bash
cd /Users/andy/.superset/worktrees/b63b1c22-154d-4a16-8535-4a258cf38d89/animated-light && git add auto-andy/skills/setup/SKILL.md && git commit -m "$(cat <<'EOF'
feat(auto-andy): Add setup skill skeleton

Add SKILL.md with frontmatter and overview for /auto-andy:setup skill.

Co-Authored-By: Claude Opus 4.6 <noreply@anthropic.com>
EOF
)"
```

---

### Task 2: Add Phase 0 - Parse Arguments

**Files:**
- Modify: `auto-andy/skills/setup/SKILL.md`

- [ ] **Step 1: Append Phase 0 to SKILL.md**

Append to `auto-andy/skills/setup/SKILL.md`:

```markdown

## Phase 0: Parse Arguments

Parse `$ARGUMENTS` for:
- `--check` - Set `CHECK_ONLY=true`
- `--force` - Set `FORCE_INSTALL=true`
- `--skip-token` - Set `SKIP_TOKEN=true`
- Default: `CHECK_ONLY=false`, `FORCE_INSTALL=false`, `SKIP_TOKEN=false`
```

- [ ] **Step 2: Verify append**

Run:
```bash
tail -15 /Users/andy/.superset/worktrees/b63b1c22-154d-4a16-8535-4a258cf38d89/animated-light/auto-andy/skills/setup/SKILL.md
```

Expected: Shows Phase 0 content.

- [ ] **Step 3: Commit**

```bash
cd /Users/andy/.superset/worktrees/b63b1c22-154d-4a16-8535-4a258cf38d89/animated-light && git add auto-andy/skills/setup/SKILL.md && git commit -m "$(cat <<'EOF'
feat(auto-andy/setup): Add argument parsing phase

Co-Authored-By: Claude Opus 4.6 <noreply@anthropic.com>
EOF
)"
```

---

### Task 3: Add Phase 1 - Detect Environment

**Files:**
- Modify: `auto-andy/skills/setup/SKILL.md`

- [ ] **Step 1: Append Phase 1 to SKILL.md**

Append to `auto-andy/skills/setup/SKILL.md`:

```markdown

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
```

- [ ] **Step 2: Verify append**

Run:
```bash
grep -c "## Phase 1" /Users/andy/.superset/worktrees/b63b1c22-154d-4a16-8535-4a258cf38d89/animated-light/auto-andy/skills/setup/SKILL.md
```

Expected: `1`

- [ ] **Step 3: Commit**

```bash
cd /Users/andy/.superset/worktrees/b63b1c22-154d-4a16-8535-4a258cf38d89/animated-light && git add auto-andy/skills/setup/SKILL.md && git commit -m "$(cat <<'EOF'
feat(auto-andy/setup): Add environment detection phase

Detects OS, shell, RC file, and package manager.

Co-Authored-By: Claude Opus 4.6 <noreply@anthropic.com>
EOF
)"
```

---

### Task 4: Add Phase 2 - Check Prerequisites

**Files:**
- Modify: `auto-andy/skills/setup/SKILL.md`

- [ ] **Step 1: Append Phase 2 to SKILL.md**

Append to `auto-andy/skills/setup/SKILL.md`:

```markdown

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
```

- [ ] **Step 2: Verify append**

Run:
```bash
grep -c "## Phase 2" /Users/andy/.superset/worktrees/b63b1c22-154d-4a16-8535-4a258cf38d89/animated-light/auto-andy/skills/setup/SKILL.md
```

Expected: `1`

- [ ] **Step 3: Commit**

```bash
cd /Users/andy/.superset/worktrees/b63b1c22-154d-4a16-8535-4a258cf38d89/animated-light && git add auto-andy/skills/setup/SKILL.md && git commit -m "$(cat <<'EOF'
feat(auto-andy/setup): Add prerequisite checking phase

Checks Go, glab, sqlite3, middleman, kata, state dir, token, and config.

Co-Authored-By: Claude Opus 4.6 <noreply@anthropic.com>
EOF
)"
```

---

### Task 5: Add Phase 3 - Install Missing Prerequisites

**Files:**
- Modify: `auto-andy/skills/setup/SKILL.md`

- [ ] **Step 1: Append Phase 3 to SKILL.md**

Append to `auto-andy/skills/setup/SKILL.md`:

```markdown

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
```

- [ ] **Step 2: Verify append**

Run:
```bash
grep -c "## Phase 3" /Users/andy/.superset/worktrees/b63b1c22-154d-4a16-8535-4a258cf38d89/animated-light/auto-andy/skills/setup/SKILL.md
```

Expected: `1`

- [ ] **Step 3: Commit**

```bash
cd /Users/andy/.superset/worktrees/b63b1c22-154d-4a16-8535-4a258cf38d89/animated-light && git add auto-andy/skills/setup/SKILL.md && git commit -m "$(cat <<'EOF'
feat(auto-andy/setup): Add installation phase

Installs Go, glab, middleman, kata via appropriate package managers.

Co-Authored-By: Claude Opus 4.6 <noreply@anthropic.com>
EOF
)"
```

---

### Task 6: Add Phase 4 - Configure Token and Middleman

**Files:**
- Modify: `auto-andy/skills/setup/SKILL.md`

- [ ] **Step 1: Append Phase 4 to SKILL.md**

Append to `auto-andy/skills/setup/SKILL.md`:

```markdown

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
```

- [ ] **Step 2: Verify append**

Run:
```bash
grep -c "## Phase 4" /Users/andy/.superset/worktrees/b63b1c22-154d-4a16-8535-4a258cf38d89/animated-light/auto-andy/skills/setup/SKILL.md
```

Expected: `1`

- [ ] **Step 3: Commit**

```bash
cd /Users/andy/.superset/worktrees/b63b1c22-154d-4a16-8535-4a258cf38d89/animated-light && git add auto-andy/skills/setup/SKILL.md && git commit -m "$(cat <<'EOF'
feat(auto-andy/setup): Add configuration phase

Configures GitLab token in shell RC and creates middleman config.toml.

Co-Authored-By: Claude Opus 4.6 <noreply@anthropic.com>
EOF
)"
```

---

### Task 7: Add Phase 5 - Verify Installation

**Files:**
- Modify: `auto-andy/skills/setup/SKILL.md`

- [ ] **Step 1: Append Phase 5 to SKILL.md**

Append to `auto-andy/skills/setup/SKILL.md`:

```markdown

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
```

- [ ] **Step 2: Verify append**

Run:
```bash
grep -c "## Phase 5" /Users/andy/.superset/worktrees/b63b1c22-154d-4a16-8535-4a258cf38d89/animated-light/auto-andy/skills/setup/SKILL.md
```

Expected: `1`

- [ ] **Step 3: Commit**

```bash
cd /Users/andy/.superset/worktrees/b63b1c22-154d-4a16-8535-4a258cf38d89/animated-light && git add auto-andy/skills/setup/SKILL.md && git commit -m "$(cat <<'EOF'
feat(auto-andy/setup): Add verification phase

Runs smoke tests on all installed components.

Co-Authored-By: Claude Opus 4.6 <noreply@anthropic.com>
EOF
)"
```

---

### Task 8: Add Phase 6 - Report Summary and Constraints

**Files:**
- Modify: `auto-andy/skills/setup/SKILL.md`

- [ ] **Step 1: Append Phase 6 and Constraints to SKILL.md**

Append to `auto-andy/skills/setup/SKILL.md`:

```markdown

## Phase 6: Report Summary

Display final summary:

```
## Auto-Andy Setup Complete

### Installed
$INSTALLED_LIST

### Configured
$CONFIGURED_LIST

### Next Steps

1. Restart your shell or run: source $RC_FILE
2. Start middleman: middleman
3. Open http://localhost:8091 and add repos via Settings
4. Run: /auto-andy:triage
```

Where:
- `$INSTALLED_LIST` lists each tool that was installed with version (e.g., "- Go 1.26.3", "- middleman (go.kenn.io/middleman@latest)")
- `$CONFIGURED_LIST` lists configuration changes (e.g., "- GitLab token added to ~/.zshrc", "- Middleman config created at ~/.config/middleman/config.toml")

**If nothing was installed/configured:**
```
All prerequisites were already installed and configured. auto-andy is ready to use.

Run: /auto-andy:triage
```

## Constraints

**Tool Restrictions:**
- **Bash:** Package manager commands (brew, apt, dnf), go install, mkdir, echo, test, which
- **Read:** Shell RC files (to check for existing token export), ~/.config/middleman/config.toml
- **Write:** Shell RC files (append token export), ~/.config/middleman/config.toml
- **AskUserQuestion:** Token prompt only

**Security:**
- Token is prompted interactively, never logged or displayed after entry
- Token is written only to user's RC file with appropriate permissions
- No tokens are hardcoded or committed
- All file writes are to user home directory only
```

- [ ] **Step 2: Verify complete skill file**

Run:
```bash
wc -l /Users/andy/.superset/worktrees/b63b1c22-154d-4a16-8535-4a258cf38d89/animated-light/auto-andy/skills/setup/SKILL.md
```

Expected: ~350-400 lines.

Run:
```bash
grep "^## Phase" /Users/andy/.superset/worktrees/b63b1c22-154d-4a16-8535-4a258cf38d89/animated-light/auto-andy/skills/setup/SKILL.md
```

Expected:
```
## Phase 0: Parse Arguments
## Phase 1: Detect Environment
## Phase 2: Check Prerequisites
## Phase 3: Install Missing Prerequisites
## Phase 4: Configure
## Phase 5: Verify Installation
## Phase 6: Report Summary
```

- [ ] **Step 3: Commit**

```bash
cd /Users/andy/.superset/worktrees/b63b1c22-154d-4a16-8535-4a258cf38d89/animated-light && git add auto-andy/skills/setup/SKILL.md && git commit -m "$(cat <<'EOF'
feat(auto-andy/setup): Add summary and constraints

Completes the setup skill with final summary output and tool constraints.

Co-Authored-By: Claude Opus 4.6 <noreply@anthropic.com>
EOF
)"
```

---

### Task 9: Bump Plugin Version

**Files:**
- Modify: `auto-andy/.claude-plugin/plugin.json`

- [ ] **Step 1: Read current plugin.json**

Run:
```bash
cat /Users/andy/.superset/worktrees/b63b1c22-154d-4a16-8535-4a258cf38d89/animated-light/auto-andy/.claude-plugin/plugin.json
```

- [ ] **Step 2: Bump version from 1.7.0 to 1.8.0**

Edit `auto-andy/.claude-plugin/plugin.json` to change version:

```json
{
  "name": "auto-andy",
  "description": "Automated MR comment triage and response using middleman, kata, and roborev",
  "version": "1.8.0",
  "author": {
    "name": "Andy Hadjigeorgiou"
  },
  "commands": "./skills"
}
```

- [ ] **Step 3: Verify version bump**

Run:
```bash
grep '"version"' /Users/andy/.superset/worktrees/b63b1c22-154d-4a16-8535-4a258cf38d89/animated-light/auto-andy/.claude-plugin/plugin.json
```

Expected: `"version": "1.8.0",`

- [ ] **Step 4: Commit**

```bash
cd /Users/andy/.superset/worktrees/b63b1c22-154d-4a16-8535-4a258cf38d89/animated-light && git add auto-andy/.claude-plugin/plugin.json && git commit -m "$(cat <<'EOF'
chore(auto-andy): Bump version to 1.8.0

Adds /auto-andy:setup skill for prerequisite installation.

Co-Authored-By: Claude Opus 4.6 <noreply@anthropic.com>
EOF
)"
```

---

### Task 10: Final Verification and Summary

**Files:**
- None (verification only)

- [ ] **Step 1: Verify skill directory structure**

Run:
```bash
ls -la /Users/andy/.superset/worktrees/b63b1c22-154d-4a16-8535-4a258cf38d89/animated-light/auto-andy/skills/
```

Expected: Shows `address/`, `post-comments/`, `setup/`, `triage/` directories.

- [ ] **Step 2: Verify skill is discoverable**

Run:
```bash
ls /Users/andy/.superset/worktrees/b63b1c22-154d-4a16-8535-4a258cf38d89/animated-light/auto-andy/skills/setup/
```

Expected: `SKILL.md`

- [ ] **Step 3: Verify git log**

Run:
```bash
cd /Users/andy/.superset/worktrees/b63b1c22-154d-4a16-8535-4a258cf38d89/animated-light && git log --oneline -10
```

Expected: Shows commits for each task.

- [ ] **Step 4: Display completion message**

```
## Implementation Complete

The /auto-andy:setup skill has been created at:
  auto-andy/skills/setup/SKILL.md

Plugin version bumped to 1.8.0.

To test:
1. Clear plugin cache: rm -rf ~/.claude/plugins/cache/marketplace/auto-andy
2. Restart Claude Code
3. Run: /auto-andy:setup --check
```
