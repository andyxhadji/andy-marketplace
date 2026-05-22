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

**Supported formats:**
- `owner/repo !123` - Post comments for a specific MR
- `--all` - Post comments for all pending MRs
- `--dry-run` - Show what would be posted without actually posting
- `--list` - List all pending approval files

**Examples:**
```
/auto-andy:post-comments rwe/foundry/llm-infra !420
/auto-andy:post-comments --all
/auto-andy:post-comments --all --dry-run
/auto-andy:post-comments --list
```

## Phase 0: Parse Arguments

Parse `$ARGUMENTS` for:
- MR reference pattern `owner/repo !number` - Set `TARGET_MR` (e.g., `rwe/foundry/llm-infra !420`)
- `--all` - Set `POST_ALL=true`
- `--dry-run` - Set `DRY_RUN=true`
- `--list` - Set `LIST_MODE=true`

**If `LIST_MODE=true`:**
```bash
find ~/.auto-andy/pending -name "mr-*.md" -type f 2>/dev/null
```
Display the list and exit.

**If neither `TARGET_MR` nor `--all` is provided:**
Display:
```
Error: Specify an MR or use --all.

Usage:
  /auto-andy:post-comments owner/repo !123   # Post for specific MR
  /auto-andy:post-comments --all             # Post all pending
  /auto-andy:post-comments --list            # List pending files
```
Exit.

## Phase 1: Discover Approval Documents

### 1.1 Find approval documents to process

**If `TARGET_MR` is set (e.g., `rwe/foundry/llm-infra !420`):**

Parse the MR reference to extract `OWNER`, `REPO`, and `MR_NUMBER`.
Construct the file path: `~/.auto-andy/pending/$OWNER/$REPO/mr-$MR_NUMBER.md`

```bash
APPROVAL_FILE=~/.auto-andy/pending/$OWNER/$REPO/mr-$MR_NUMBER.md
test -f "$APPROVAL_FILE" && echo "exists" || echo "missing"
```

**If `POST_ALL=true`:**

Find all pending approval files:
```bash
find ~/.auto-andy/pending -name "mr-*.md" -type f
```

Set `APPROVAL_FILES` to the list of found files.

**If no files found:**
Display:
```
No pending approval documents found.

Nothing to post. Run /auto-andy:address --auto first to generate pending approvals.
```
Exit.

### 1.2 Read approval documents

For each approval file, read its contents.

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

### 1.4 Display and confirm comments

**Display each comment with original and response side-by-side:**

For each proposed comment, display:
```
---
### $PROJECT MR !$MR_NUMBER (@$AUTHOR)

**Original comment:**
> $ORIGINAL_COMMENT

**Proposed response:**
$PROPOSED_RESPONSE
---
```

**If `DRY_RUN=false`:**

After displaying all comments, ask:
```
The above $COMMENT_COUNT comments will be posted.

Type 'yes' to proceed, or 'no' to cancel:
```

**If user responds 'no' or anything other than 'yes':**
Display:
```
Cancelled. Edit ~/.auto-andy/pending-approval.md and run again when ready.
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

**Step 1: Look up the discussion_id for the original note**

GitLab has two types of MR comments:
- **Regular notes**: General MR comments (use `in_reply_to_id`)
- **DiffNotes**: Code review comments on specific lines (require posting to the discussion endpoint)

To reply correctly, first look up the discussion containing the original note:

```bash
glab api "projects/$PROJECT_ID/merge_requests/$MR_NUMBER/discussions" | \
  jq -r '.[] | select(.notes[].id == '$ORIGINAL_NOTE_ID') | .id'
```

This returns the `DISCUSSION_ID` if the note is part of a discussion thread.

**Step 2: Post the reply**

**If DISCUSSION_ID was found (DiffNote/code review comment):**

Post to the discussions endpoint to create a threaded reply:

```bash
glab api -X POST \
  "projects/$PROJECT_ID/merge_requests/$MR_NUMBER/discussions/$DISCUSSION_ID/notes" \
  -f body="$RESPONSE_TEXT"
```

**If no DISCUSSION_ID (regular MR note):**

Fall back to the notes endpoint with `in_reply_to_id`:

```bash
curl -s -X POST \
  "https://$PLATFORM_HOST/api/v4/projects/$PROJECT_ID/merge_requests/$MR_NUMBER/notes" \
  -H "PRIVATE-TOKEN: $GITLAB_TOKEN" \
  -H "Content-Type: application/json" \
  -d "{\"body\": \"$ESCAPED_RESPONSE_TEXT\", \"in_reply_to_id\": $ORIGINAL_NOTE_ID}"
```

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

## Phase 5: Archive Approval Documents

**If `DRY_RUN=false` and at least one comment was posted:**

For each approval file that was fully processed (all comments posted successfully):

### 5.1 Create archive directory

```bash
ARCHIVE_DIR=~/.auto-andy/history/$(date +"%Y-%m-%d-%H%M%S")
mkdir -p "$ARCHIVE_DIR/$OWNER/$REPO"
```

### 5.2 Move approval doc to archive

```bash
mv ~/.auto-andy/pending/$OWNER/$REPO/mr-$MR_NUMBER.md "$ARCHIVE_DIR/$OWNER/$REPO/mr-$MR_NUMBER.md"
```

### 5.3 Clean up empty directories

```bash
rmdir ~/.auto-andy/pending/$OWNER/$REPO 2>/dev/null || true
rmdir ~/.auto-andy/pending/$OWNER 2>/dev/null || true
```

### 5.4 If some comments failed

**If `FAILED_COUNT > 0` for an MR:**

Keep the approval file in place but update it:
- Remove successfully posted comments
- Keep only the comments that failed to post
- Add note in header: "Partial post - contains only failed comments"

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
- **Read:** `~/.auto-andy/pending/**/*.md`
- **Write:** `~/.auto-andy/pending/**/*.md` (for failed-only regeneration)
- **AskUserQuestion:** For confirmation before posting

**Security:**
- API tokens read from environment variables only
- middleman DB is read-only (SELECT queries only)
- No modification of code or repo files
