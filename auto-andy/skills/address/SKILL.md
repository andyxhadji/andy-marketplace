---
name: address
description: Process kata tasks from MRs, auto-address high-confidence tasks, create spec handoffs for complex ones
allowed-tools: [Bash, Read, Write, Edit, Glob, Grep, Agent, AskUserQuestion, Skill(roborev-refine)]
---

# Address MR Comment Tasks

## Objective

Process kata tasks labeled `auto-andy`, analyze each for complexity/confidence, auto-address high-confidence tasks (with roborev-refine for quality), and create spec handoff documents for low-confidence tasks. Generate an approval document for human review.

## Arguments

The user's argument (if any) is: $ARGUMENTS

**Supported flags:**
- `--auto` - Required flag to run automated addressing
- `--project <name>` - Process only the specified project
- `--dry-run` - Analyze tasks without making changes
- `--reset` - Clear pending-approval.md and start fresh

## Phase 0: Parse Arguments and Validate

### 0.1 Parse flags

Parse `$ARGUMENTS` for:
- `--auto` - Set `AUTO_MODE=true`
- `--project <name>` - Set `TARGET_PROJECT` variable
- `--dry-run` - Set `DRY_RUN=true`
- `--reset` - Set `RESET_MODE=true`

**If `--auto` is not present:**
Display:
```
Error: The --auto flag is required to run automated addressing.

Usage:
  /auto-andy:address --auto              # Process all projects
  /auto-andy:address --auto --project ml-llms  # Single project
  /auto-andy:address --auto --dry-run    # Preview without changes
```
Exit.

### 0.2 Handle reset mode

**If `RESET_MODE=true`:**
```bash
rm -rf ~/.auto-andy/pending/
mkdir -p ~/.auto-andy/pending
echo "Cleared all pending approval documents"
```

## Phase 1: Ensure State Directory

Run:
```bash
mkdir -p ~/.auto-andy/pending
mkdir -p ~/.auto-andy/specs
mkdir -p ~/.auto-andy/history
```

## Phase 2: Discover Ready, Unowned Tasks

### 2.1 Query kata for ready, unowned auto-andy tasks

Use `kata ready --unowned` to find tasks that are:
- Ready to work on (no open blocking predecessors)
- Not owned by any actor (available to claim)

Run:
```bash
kata ready --unowned --label auto-andy --json
```

**If `TARGET_PROJECT` is set:**
```bash
kata ready --project "$TARGET_PROJECT" --unowned --label auto-andy --json
```

### 2.2 Parse and group by project

Parse JSON output to extract tasks. Group tasks by project name.

**If no tasks found:**
Display:
```
No ready, unowned tasks with 'auto-andy' label found.
Nothing to process.

Hint: Tasks may be blocked, already owned, or not exist.
  - Check all auto-andy tasks: kata list --label auto-andy --state open
  - Check owned tasks: kata list --label auto-andy --owner $USER
```
Exit with success.

### 2.3 Display task summary

```
## Tasks to Process

| Project | Task ID | Title |
|---------|---------|-------|
| ml-llms | #42 | Fix typo in config |
| ml-llms | #43 | Add validation check |
| llm-infra | #17 | Refactor auth module |

Total: $TASK_COUNT tasks across $PROJECT_COUNT projects
```

## Phase 3: Process Tasks by Project

For each project with tasks, spawn a subagent using the Agent tool:

**Subagent prompt:**
```
Process the following kata tasks for project '$PROJECT_NAME'.

For each task:
1. Read the task details from kata
2. Analyze the MR comment for complexity and your confidence in addressing it
3. If confidence > 80%: make the changes, run tests, invoke roborev-refine, commit, push
4. If confidence <= 80%: create a spec handoff document

Return a structured result for each task.

Tasks to process:
$TASK_LIST

Working directory for this project: [determine from middleman or local clone]
```

**Subagent task processing (executed by each subagent):**

For each task:

#### 3.1 Claim the task

Before processing, claim ownership to prevent other agents from picking it up:

```bash
kata claim $TASK_ID --comment "Claimed by auto-andy for automated processing"
```

**If claim fails (already owned):**
- Skip this task
- Log: `Skipping task #$TASK_ID - already claimed by another actor`
- Continue to next task

#### 3.2 Get task details

```bash
kata show $TASK_ID --json
```

Parse the body to extract:
- `MR_URL`: URL of the merge request
- `ORIGINAL_COMMENT`: The reviewer's comment
- `DEDUPE_KEY`: For tracking
- `HEAD_BRANCH`: Branch to work on
- `AUTHOR`: Comment author

#### 3.3 Analyze complexity and confidence

Analyze the comment and determine:
- **Complexity score (1-5):**
  - 1: Trivial (typo, formatting)
  - 2: Simple (rename, add import)
  - 3: Moderate (refactor small function, add validation)
  - 4: Complex (new feature, significant refactor)
  - 5: Major (architectural change, cross-cutting concern)

- **Confidence (0-100%):**
  - How confident are you that you can correctly address this comment?
  - Consider: clarity of request, scope, dependencies, ambiguity

#### 3.4 Branch based on confidence

**If confidence > 80% (HIGH CONFIDENCE):**

a. Check out the MR branch:
```bash
cd $PROJECT_PATH
git fetch origin $HEAD_BRANCH
git checkout $HEAD_BRANCH
git pull origin $HEAD_BRANCH
```

b. Make the required changes using Edit/Write tools

c. Run tests:
```bash
# Project-specific test command - detect from repo
pytest tests/ -v  # or npm test, make test, etc.
```

d. **If tests fail:**
   - Revert changes: `git checkout -- .`
   - Mark task as `needs-human`
   - Create spec handoff instead
   - Continue to next task

e. Invoke roborev-refine:
```
Skill(roborev-refine)
```

f. **If roborev-refine exceeds 3 iterations:**
   - Mark task as `needs-human`
   - Create spec handoff instead
   - Continue to next task

g. Commit with kata task reference:
```bash
git add -A
git commit -m "$(cat <<'EOF'
fix: Address review comment

Addresses kata task #$TASK_ID

$BRIEF_DESCRIPTION

Co-Authored-By: Claude Opus 4.6 <noreply@anthropic.com>
EOF
)"
```

h. Push:
```bash
git push origin $HEAD_BRANCH
```

i. Get commit SHA:
```bash
git rev-parse HEAD
```

j. Update kata task:
```bash
kata label $TASK_ID --add addressed
kata comment $TASK_ID "Addressed in commit $COMMIT_SHA"
```

k. Draft response for approval doc:
- If change was trivial (complexity 1-2): Brief response like "Fixed in $COMMIT_SHA."
- If change was substantive (complexity 3+): Detailed response explaining what was changed and why.

l. Record result:
```
RESULT: addressed
PROJECT: $PROJECT_NAME
TASK_ID: $TASK_ID
COMMIT: $COMMIT_SHA
RESPONSE: $DRAFTED_RESPONSE
DEDUPE_KEY: $DEDUPE_KEY
MR_URL: $MR_URL
```

**If confidence <= 80% (LOW CONFIDENCE):**

a. Create spec directory:
```bash
mkdir -p ~/.auto-andy/specs/$PROJECT_NAME
```

b. Write spec handoff document to `~/.auto-andy/specs/$PROJECT_NAME/$TASK_ID-spec.md`:
```markdown
# Spec Handoff: Task #$TASK_ID

## Problem Statement

$ORIGINAL_COMMENT

## MR Context

- **MR URL:** $MR_URL
- **Branch:** $HEAD_BRANCH
- **Comment Author:** @$AUTHOR

## Why Confidence is Low

$EXPLANATION_OF_LOW_CONFIDENCE

Examples:
- Ambiguous scope: "refactor this" without clear boundaries
- External dependencies: requires changes to other repos
- Unclear intent: comment could be interpreted multiple ways
- High risk: changes could break other functionality

## Surrounding Code Context

$RELEVANT_CODE_SNIPPETS

## Suggested Next Steps

1. Clarify the exact requirement with the reviewer
2. [Additional context-specific steps]
3. Use superpowers:brainstorming to develop a full spec

---
Created by auto-andy on $(date -u +"%Y-%m-%d %H:%M:%S")
```

c. Update kata task:
```bash
kata label $TASK_ID --add needs-spec
kata comment $TASK_ID "Spec handoff created at ~/.auto-andy/specs/$PROJECT_NAME/$TASK_ID-spec.md"
```

d. Record result:
```
RESULT: spec-created
PROJECT: $PROJECT_NAME
TASK_ID: $TASK_ID
SPEC_PATH: ~/.auto-andy/specs/$PROJECT_NAME/$TASK_ID-spec.md
REASON: $LOW_CONFIDENCE_REASON
```

## Phase 4: Generate Approval Documents (Per-MR)

Pending approval documents are namespaced by MR to allow independent review and posting.

### 4.1 Group results by MR

Group all processed tasks by their MR (using `$OWNER/$REPO` and `$MR_NUMBER` from the dedupe_key).

### 4.2 For each MR, create/update approval document

For each unique MR:

**Determine file path:**
```
~/.auto-andy/pending/$OWNER/$REPO/mr-$MR_NUMBER.md
```

Example: `~/.auto-andy/pending/rwe/foundry/llm-infra/mr-420.md`

**Create directory if needed:**
```bash
mkdir -p ~/.auto-andy/pending/$OWNER/$REPO
```

**Check for existing approval doc for this MR:**
Read `~/.auto-andy/pending/$OWNER/$REPO/mr-$MR_NUMBER.md` if it exists.

**Write/update the approval document:**

```markdown
# Auto-Andy Pending Approval

**MR:** $OWNER/$REPO !$MR_NUMBER
**Branch:** $HEAD_BRANCH
**Generated:** $(date -u +"%Y-%m-%d %H:%M:%S")

## Changes Made

| Task | Commit | Response |
|------|--------|----------|
$CHANGES_TABLE_ROWS

## Specs Started (Need Manual Review)

| Task | Spec Path | Reason |
|------|-----------|--------|
$SPECS_TABLE_ROWS

## Proposed MR Comments

$PROPOSED_COMMENTS_SECTION

---

**To post these comments after review:**
```
/auto-andy:post-comments $OWNER/$REPO !$MR_NUMBER
```
```

Where `$PROPOSED_COMMENTS_SECTION` contains for each addressed task in this MR:

```markdown
### Comment by @$AUTHOR
<!-- dedupe_key: $DEDUPE_KEY -->
<!-- kata_task: $TASK_ID -->
> Original: "$ORIGINAL_COMMENT_TRUNCATED"

**Proposed response:**
$DRAFTED_RESPONSE

---
```

### 4.3 If appending to existing MR doc

If the MR's approval doc already exists (and `--reset` not used):
- Read existing content
- Append new entries to each section
- Deduplicate by task ID (skip if task already present)

## Phase 5: Output Summary

Display:
```
## Address Summary

**Mode:** $MODE_DESCRIPTION
**Projects processed:** $PROJECT_COUNT

### Results

| Outcome | Count |
|---------|-------|
| Tasks addressed | $ADDRESSED_COUNT |
| Specs created | $SPECS_COUNT |
| Skipped (already claimed) | $CLAIMED_COUNT |
| Skipped (stale MR) | $STALE_COUNT |
| Failed | $FAILED_COUNT |

### Next Steps

1. Review pending approvals: `ls ~/.auto-andy/pending/*/*/`
2. Edit responses as needed in the per-MR files
3. Post for a specific MR: `/auto-andy:post-comments owner/repo !123`
4. Or post all pending: `/auto-andy:post-comments --all`

$DRY_RUN_NOTE
```

## Constraints

**Tool Restrictions:**
- **Bash:** Git operations, kata CLI, test commands, mkdir
- **Read:** Task bodies, code files, existing approval doc
- **Write:** `~/.auto-andy/pending/**/*.md`, `~/.auto-andy/specs/**`
- **Edit:** Code files in project directories only
- **Agent:** For parallel project processing
- **Skill(roborev-refine):** For code quality validation

**Security:**
- Only modify files in the MR branch being addressed
- Only write to ~/.auto-andy/ state directory
- kata operations through official CLI only
