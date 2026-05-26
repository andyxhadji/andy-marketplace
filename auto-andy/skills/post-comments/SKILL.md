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

### 1.4 Fetch commit changes for validation

For each unique commit SHA mentioned in the approval document's "Changes Made" table:

**Step 1: Identify the repo and commit**

Parse the `<!-- dedupe_key: ... -->` to determine the repo location. Query middleman for the local worktree path:

```bash
sqlite3 ~/.config/middleman/middleman.db "
SELECT pw.worktree_path
FROM middleman_project_worktrees pw
JOIN middleman_repos r ON pw.repo_id = r.id
WHERE r.owner = '\$OWNER'
  AND r.name = '\$REPO'
  AND pw.branch = '\$HEAD_BRANCH';
"
```

**Step 2: Get the diff for the commit**

```bash
cd "\$WORKTREE_PATH" && git show \$COMMIT_SHA --stat --format="%h %s"
```

Store the diff summary as `COMMIT_CHANGES` for display.

**Step 3: For DiffNote comments, get the specific file diff**

If the original comment was on a specific file (DiffNote), extract the relevant file changes:

```bash
cd "\$WORKTREE_PATH" && git show \$COMMIT_SHA -- "\$FILE_PATH"
```

Store as `FILE_DIFF` for that comment.

### 1.5 Display and confirm comments

**Display each comment with original, changes, and response:**

For each proposed comment, display:
```
---
### $PROJECT MR !$MR_NUMBER (@$AUTHOR)

**Original comment:**
> $ORIGINAL_COMMENT

**Commit changes ($COMMIT_SHA):**
\`\`\`diff
$FILE_DIFF_OR_COMMIT_SUMMARY
\`\`\`

**Proposed response:**
$PROPOSED_RESPONSE
---
```

The commit changes section helps the user validate that:
1. The proposed response accurately describes what was changed
2. The changes actually address the reviewer's comment
3. No discrepancies exist between the response and the actual diff

**If `DRY_RUN=false`:**

After displaying all comments, ask:
```
The above $COMMENT_COUNT comments will be posted.

Review the diffs to ensure responses match the actual changes.

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

### 2.2 Verify middleman is running

```bash
curl -s http://127.0.0.1:8091/api/v1/health 2>/dev/null || echo "middleman not running"
```

**If middleman is not running:**
Display error:
```
Error: middleman is not running.

Start middleman first, or check `middleman status`.
```
Exit.

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

### 3.2 Look up discussion_id from middleman

Query middleman's database for the `thread_id` (discussion_id) of the original note:

```bash
sqlite3 ~/.config/middleman/middleman.db "
SELECT thread_id
FROM middleman_mr_events
WHERE dedupe_key = '$DEDUPE_KEY';
"
```

The `thread_id` is a 40-character hex string (e.g., `a1b2c3d4e5f6...`).

**If thread_id is NULL or empty:**

This is a top-level MR comment without a discussion thread. Post a new top-level comment:

```bash
curl -s -X POST \
  "http://127.0.0.1:8091/api/v1/providers/$PLATFORM/repos/$OWNER/$REPO/pulls/$MR_NUMBER/comments" \
  -H "Content-Type: application/json" \
  -d '{"body": "'"$ESCAPED_RESPONSE_TEXT"'"}'
```

### 3.3 Post reply via middleman

**If thread_id was found:**

Use middleman's discussion reply endpoint:

```bash
curl -s -X POST \
  "http://127.0.0.1:8091/api/v1/providers/$PLATFORM/repos/$OWNER/$REPO/pulls/$MR_NUMBER/discussions/$THREAD_ID/reply" \
  -H "Content-Type: application/json" \
  -d '{"body": "'"$ESCAPED_RESPONSE_TEXT"'"}'
```

This works for both GitLab and GitHub - middleman handles the platform-specific API calls.

**Check response:**
- HTTP 201 with `"id":` in body - success
- HTTP 4xx/5xx or `"error":` - failure

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
- **Bash:** curl for middleman API calls, kata CLI, sqlite3 queries, file operations
- **Read:** `~/.auto-andy/pending/**/*.md`
- **Write:** `~/.auto-andy/pending/**/*.md` (for failed-only regeneration)
- **AskUserQuestion:** For confirmation before posting

**Prerequisites:**
- middleman must be running on `127.0.0.1:8091`
- middleman handles authentication via its configured tokens

**Security:**
- All API calls go through middleman (no direct platform API calls)
- middleman DB is read-only (SELECT queries only)
- No modification of code or repo files
