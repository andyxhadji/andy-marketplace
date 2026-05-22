---
name: triage
description: Query middleman for new MR comments and create kata tasks for actionable items
allowed-tools: [Bash, Read, Write, AskUserQuestion]
---

# Triage MR Comments into Kata Tasks

## Objective

Query middleman's SQLite database for new MR comments on **your own MRs/PRs** since last triage, identify actionable comments (explicit requests/questions), and create kata tasks for each.

## Arguments

The user's argument (if any) is: $ARGUMENTS

**Supported flags:**
- `--repo <name>` - Process only the specified repo (e.g., `--repo ml-llms`)
- `--dry-run` - Show what would be created without creating tasks
- No flags - Process all repos in middleman

## Phase 0: Parse Arguments

Parse `$ARGUMENTS` for:
- `--repo <name>` - Extract repo name, set `TARGET_REPO` variable
- `--dry-run` - Set `DRY_RUN=true`
- Default: `TARGET_REPO=all`, `DRY_RUN=false`

## Phase 1: Verify Prerequisites

### 1.1 Get current user's username

Run:
```bash
git config user.email | sed 's/@.*//'
```

Store the result as `CURRENT_USER`. This extracts the username portion of the git email (e.g., `andy@flatiron.com` → `andy`).

**If empty or fails:** Display error and exit:
```
Error: Could not determine current user from git config.
Please ensure git user.email is configured.
```

### 1.2 Check middleman is running

Run:
```bash
middleman status 2>&1
```

**If output contains "running":** Continue to Phase 2.

**If output contains error or "not running":**
Display error and exit:
```
Error: middleman is not running.

To start middleman:
  middleman

To check status:
  middleman status
```

### 1.3 Ensure state directory exists

Run:
```bash
mkdir -p ~/.auto-andy
```

## Phase 2: Get Last Triage Timestamp

### 2.1 Read timestamp file

Run:
```bash
cat ~/.auto-andy/last-triage.txt 2>/dev/null || echo ""
```

**If file exists and has content:** Use that timestamp as `SINCE_TIMESTAMP`.

**If file empty or doesn't exist:** Calculate 24 hours ago:
```bash
date -u -v-24H +"%Y-%m-%d %H:%M:%S"
```
Use result as `SINCE_TIMESTAMP`.

## Phase 3: Query Middleman for New Comments

### 3.1 Build and execute SQL query

Run:
```bash
sqlite3 ~/.config/middleman/middleman.db "
SELECT
    e.id,
    e.dedupe_key,
    e.author,
    e.body,
    e.created_at,
    e.platform_external_id,
    mr.number as mr_number,
    mr.title as mr_title,
    mr.url as mr_url,
    mr.head_branch,
    r.name as repo_name,
    r.owner as repo_owner,
    r.platform,
    r.platform_host
FROM middleman_mr_events e
JOIN middleman_merge_requests mr ON e.merge_request_id = mr.id
JOIN middleman_repos r ON mr.repo_id = r.id
WHERE e.event_type = 'issue_comment'
  AND e.created_at > '\$SINCE_TIMESTAMP'
  AND mr.state = 'open'
  AND mr.author = '\$CURRENT_USER'
ORDER BY e.created_at;
"
```

**If `TARGET_REPO` is not "all":** Add `AND r.name = '\$TARGET_REPO'` to WHERE clause.

### 3.2 Parse results

For each row returned, extract:
- `comment_id`: e.id
- `dedupe_key`: e.dedupe_key
- `author`: e.author
- `body`: e.body (the comment text)
- `platform_external_id`: e.platform_external_id (the note/comment ID on the platform)
- `mr_number`: mr.number
- `mr_title`: mr.title
- `mr_url`: mr.url
- `head_branch`: mr.head_branch
- `repo_name`: r.name
- `repo_owner`: r.owner
- `platform`: r.platform
- `platform_host`: r.platform_host

**Construct comment URL based on platform:**
- GitLab: `$MR_URL#note_$PLATFORM_EXTERNAL_ID`
- GitHub: `$MR_URL#issuecomment-$PLATFORM_EXTERNAL_ID`

Store as `COMMENT_URL`.

**If no rows returned:**
Display:
```
Triage complete: 0 new comments found since $SINCE_TIMESTAMP.
No tasks created.
```
Skip to Phase 6.

## Phase 4: Filter for Actionable Comments

For each comment, determine if it's actionable using these criteria:

**ACTIONABLE (create task):**
- Contains question directed at author: `?` with context suggesting request
- Contains explicit request words: "please", "can you", "could you", "should", "consider", "fix", "change", "update", "add", "remove", "rename"
- Contains review feedback requiring action: "nit:", "suggestion:", "todo:", "blocking:"

**NOT ACTIONABLE (skip):**
- Pure acknowledgments: "LGTM", "looks good", "thanks", "approved", "+1", "ship it"
- Bot comments (author contains "bot" or "[bot]")
- Self-replies (would need to check if author is MR author - skip this check for simplicity)
- Questions that are rhetorical or not requiring action

**Decision process:**
Analyze each comment body. If uncertain, err on the side of creating a task (false positives are easier to close than missed tasks).

Track counts:
- `TOTAL_COMMENTS`: Total comments processed
- `ACTIONABLE_COMMENTS`: Comments identified as actionable
- `SKIPPED_COMMENTS`: Comments skipped as not actionable

## Phase 5: Create Kata Tasks

For each actionable comment:

### 5.1 Check if task already exists

Run:
```bash
kata search --project "$REPO_NAME" "$DEDUPE_KEY" --json 2>/dev/null
```

**If search returns results containing the dedupe_key:** Skip this comment (already processed).

### 5.2 Create kata task

**If `DRY_RUN=true`:**
Display what would be created:
```
[DRY-RUN] Would create task in project '$REPO_NAME':
  Title: $EXTRACTED_TITLE
  Labels: from-mr, auto-andy
  Body preview: $BODY_PREVIEW...
```
Continue to next comment.

**If `DRY_RUN=false`:**

Extract a title from the comment:
- If comment is short (<80 chars): use full comment as title
- If comment starts with "nit:", "suggestion:", etc.: use that prefix + first phrase
- Otherwise: extract first sentence or first 80 chars

Create the task (note: title is a positional argument, not a flag):
```bash
kata create "$EXTRACTED_TITLE" --project "$REPO_NAME" --label from-mr --label auto-andy --body "$(cat <<'BODY'
## MR Comment

**From:** @$AUTHOR
**MR:** $MR_TITLE ($MR_URL)
**Branch:** $HEAD_BRANCH
**Comment:** $COMMENT_URL

### Original Comment

$BODY

---

**Dedupe Key:** \`$DEDUPE_KEY\`
**Platform:** $PLATFORM ($PLATFORM_HOST)
BODY
)"
```

Track: `TASKS_CREATED` counter.

**If kata create fails (project not found):**
Display warning:
```
Warning: Kata project '$REPO_NAME' not found. Skipping comment.
Hint: Create project with: kata init --project $REPO_NAME
```
Track: `SKIPPED_NO_PROJECT` counter.

## Phase 6: Update Timestamp

**If `DRY_RUN=false`:**

Write current timestamp:
```bash
date -u +"%Y-%m-%d %H:%M:%S" > ~/.auto-andy/last-triage.txt
```

## Phase 7: Output Summary

Display summary:
```
## Triage Summary

**User:** $CURRENT_USER
**Time range:** $SINCE_TIMESTAMP to now
**Repos processed:** $REPOS_PROCESSED

| Metric | Count |
|--------|-------|
| Comments scanned | $TOTAL_COMMENTS |
| Actionable | $ACTIONABLE_COMMENTS |
| Skipped (not actionable) | $SKIPPED_COMMENTS |
| Tasks created | $TASKS_CREATED |
| Skipped (already exists) | $SKIPPED_EXISTING |
| Skipped (no kata project) | $SKIPPED_NO_PROJECT |

$DRY_RUN_NOTE
```

Where `$DRY_RUN_NOTE` is:
- If dry-run: "**Note:** Dry-run mode - no tasks were actually created."
- Otherwise: empty

## Constraints

**Tool Restrictions:**
- **Bash:** SQLite queries (read-only), kata CLI, date commands, mkdir
- **Read:** `~/.auto-andy/last-triage.txt` only
- **Write:** `~/.auto-andy/last-triage.txt` only
- **AskUserQuestion:** Not used in normal flow (skill is non-interactive)

**Security:**
- Middleman DB is read-only (SELECT queries only)
- No modification of middleman data
- kata tasks created through official CLI
