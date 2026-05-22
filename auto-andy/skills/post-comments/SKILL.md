---
name: post-comments
description: Post approved responses to MRs from pending-approval.md
allowed-tools: [Bash, Read, Write, AskUserQuestion]
---

# Post Approved Comments to MRs

## Objective

Read the approved `pending-approval.md`, post responses to the original MR comments via GitLab/GitHub API, update kata tasks, and archive the approval document.

## Arguments

The user's argument (if any) is: $ARGUMENTS

**Supported flags:**
- `--dry-run` - Show what would be posted without actually posting
- No flags - Post all approved comments

## Phase 0: Parse Arguments

Parse `$ARGUMENTS` for:
- `--dry-run` - Set `DRY_RUN=true`
- Default: `DRY_RUN=false`

## Phase 1: Verify Prerequisites

### 1.1 Check approval doc exists

Run:
```bash
test -f ~/.auto-andy/pending-approval.md && echo "exists" || echo "missing"
```

**If "missing":**
Display:
```
No pending approval document found at ~/.auto-andy/pending-approval.md

Nothing to post. Run /auto-andy:address --auto first to generate pending approvals.
```
Exit.

### 1.2 Read approval document

Read `~/.auto-andy/pending-approval.md`.

### 1.3 Parse proposed comments

Extract each proposed comment section:
- Parse `<!-- dedupe_key: ... -->` for API targeting
- Parse `<!-- kata_task: ... -->` for task updates
- Extract `**Proposed response:**` content as the comment body
- Extract project name, MR number, author from header

**If no proposed comments section found:**
Display:
```
No proposed comments found in pending-approval.md.
Nothing to post.
```
Exit.

### 1.4 Confirm user has reviewed

**If `DRY_RUN=false`:**

Ask:
```
Have you reviewed and edited ~/.auto-andy/pending-approval.md?

The following comments will be posted:

$COMMENT_PREVIEW_LIST

Type 'yes' to proceed, or 'no' to cancel:
```

**If user responds 'no' or anything other than 'yes':**
Display:
```
Cancelled. Review the file and run again when ready.
```
Exit.

## Phase 2: Resolve API Details

For each comment to post:

### 2.1 Parse dedupe_key

Dedupe key format: `$PLATFORM:$PLATFORM_HOST:$OWNER/$REPO:mr:$MR_NUMBER:note:$NOTE_ID`

Example: `gitlab:git.the.flatiron.com:agentic-ai/plugin-marketplace:mr:62:note:2474284`

Extract:
- `PLATFORM`: gitlab or github
- `PLATFORM_HOST`: e.g., git.the.flatiron.com or github.com
- `OWNER`: repo owner
- `REPO`: repo name
- `MR_NUMBER`: merge request number
- `ORIGINAL_NOTE_ID`: the note ID to reply to (e.g., 2474284)

### 2.2 Get API token

**For GitLab (platform = "gitlab"):**
```bash
echo $MIDDLEMAN_GITLAB_FLATIRON_TOKEN
```

**For GitHub (platform = "github"):**
```bash
echo $MIDDLEMAN_GITHUB_TOKEN
```

**If token is empty:**
Display error:
```
Error: No API token found for $PLATFORM.

Set the environment variable:
- GitLab: export MIDDLEMAN_GITLAB_FLATIRON_TOKEN=<your-token>
- GitHub: export MIDDLEMAN_GITHUB_TOKEN=<your-token>
```
Exit.

### 2.3 Get project ID for GitLab

**For GitLab only:**

Query middleman for project ID:
```bash
sqlite3 ~/.config/middleman/middleman.db "
SELECT platform_repo_id
FROM middleman_repos
WHERE platform = 'gitlab'
  AND platform_host = '$PLATFORM_HOST'
  AND owner = '$OWNER'
  AND name = '$REPO';
"
```

## Phase 3: Post Comments

For each proposed comment:

### 3.1 Dry-run mode

**If `DRY_RUN=true`:**
Display:
```
[DRY-RUN] Would post to $PLATFORM $OWNER/$REPO MR !$MR_NUMBER:
---
$RESPONSE_TEXT
---
```
Continue to next comment.

### 3.2 Post to GitLab

**If `PLATFORM=gitlab`:**

Post as a reply to the original comment thread using `in_reply_to_id`:

```bash
curl -s -X POST \
  "https://$PLATFORM_HOST/api/v4/projects/$PROJECT_ID/merge_requests/$MR_NUMBER/notes" \
  -H "PRIVATE-TOKEN: $GITLAB_TOKEN" \
  -H "Content-Type: application/json" \
  -d "{\"body\": \"$ESCAPED_RESPONSE_TEXT\", \"in_reply_to_id\": $ORIGINAL_NOTE_ID}"
```

This ensures the response appears as a threaded reply to the reviewer's original comment, not as a standalone MR comment.

**Check response:**
- If response contains `"id":` - success
- If response contains `"error":` or HTTP error - failure

### 3.3 Post to GitHub

**If `PLATFORM=github`:**

For PR review comments, post as a reply to the review thread:

```bash
curl -s -X POST \
  "https://api.github.com/repos/$OWNER/$REPO/pulls/$MR_NUMBER/comments/$ORIGINAL_NOTE_ID/replies" \
  -H "Authorization: Bearer $GITHUB_TOKEN" \
  -H "Accept: application/vnd.github+json" \
  -d "{\"body\": \"$ESCAPED_RESPONSE_TEXT\"}"
```

This ensures the response appears as a threaded reply to the reviewer's original comment.

**Check response:**
- If response contains `"id":` - success
- If response contains `"message":` with error - failure

### 3.4 Handle result

**On success:**
- Track: `POSTED_COUNT++`
- Record comment URL from response

**On failure:**
- Track: `FAILED_COUNT++`
- Record error message
- Continue to next comment (don't stop on individual failures)

## Phase 4: Update Kata Tasks

For each successfully posted comment:

### 4.1 Add responded label

```bash
kata label $KATA_TASK_ID --add responded
```

### 4.2 Close task with comment

```bash
kata close $KATA_TASK_ID --comment "Response posted to MR: $COMMENT_URL"
```

## Phase 5: Archive Approval Document

**If `DRY_RUN=false` and at least one comment was posted:**

### 5.1 Create archive directory

```bash
ARCHIVE_DIR=~/.auto-andy/history/$(date +"%Y-%m-%d-%H%M%S")
mkdir -p "$ARCHIVE_DIR"
```

### 5.2 Move approval doc to archive

```bash
mv ~/.auto-andy/pending-approval.md "$ARCHIVE_DIR/approval.md"
```

### 5.3 If some comments failed

**If `FAILED_COUNT > 0`:**

Create new pending-approval.md with only the failed comments:
- Copy header
- Include only the comments that failed to post
- Note in header: "Regenerated after partial post - contains only failed comments"

## Phase 6: Output Summary

Display:
```
## Post Comments Summary

| Metric | Count |
|--------|-------|
| Comments posted | $POSTED_COUNT |
| Comments failed | $FAILED_COUNT |
| Tasks closed | $TASKS_CLOSED |

$FAILURE_DETAILS

$ARCHIVE_NOTE

$DRY_RUN_NOTE
```

Where:
- `$FAILURE_DETAILS`: If any failures, list each with error message
- `$ARCHIVE_NOTE`: If archived, "Approval doc archived to $ARCHIVE_DIR/approval.md"
- `$DRY_RUN_NOTE`: If dry-run, "**Note:** Dry-run mode - no comments were actually posted."

## Constraints

**Tool Restrictions:**
- **Bash:** curl for API calls, kata CLI, sqlite3 queries, file operations
- **Read:** `~/.auto-andy/pending-approval.md` only
- **Write:** `~/.auto-andy/pending-approval.md` (for failed-only regeneration)
- **AskUserQuestion:** For confirmation before posting

**Security:**
- API tokens read from environment variables only
- middleman DB is read-only (SELECT queries only)
- No modification of code or repo files
