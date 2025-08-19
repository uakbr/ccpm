---
allowed-tools: Bash, Read, Write, LS
---

# Epic Sync

Push epic and tasks to GitHub as issues.

## Usage
```
/pm:epic-sync <feature_name>
```

## Quick Check

```bash
# Verify epic exists
test -f .claude/epics/$ARGUMENTS/epic.md || echo "❌ Epic not found. Run: /pm:prd-parse $ARGUMENTS"

# Count task files
ls .claude/epics/$ARGUMENTS/*.md 2>/dev/null | grep -v epic.md | wc -l
```

If no tasks found: "❌ No tasks to sync. Run: /pm:epic-decompose $ARGUMENTS"

## Instructions

### 1. Create Epic Issue

Read epic content and create GitHub issue:
```bash
gh issue create \
  --title "Epic: $ARGUMENTS" \
  --body-file .claude/epics/$ARGUMENTS/epic.md \
  --label "epic,epic:$ARGUMENTS"
```

Store the returned issue number for epic frontmatter update.

### 2. Create Task Sub-Issues

Check if gh-sub-issue is available:
```bash
if gh extension list | grep -q "yahsan2/gh-sub-issue"; then
  use_subissues=true
else
  use_subissues=false
  echo "⚠️ gh-sub-issue not installed. Using fallback mode."
fi
```

For each numbered task file (001.md, 002.md, etc.):

**With gh-sub-issue (preferred):**
```bash
if [ "$use_subissues" = true ]; then
  task_number=$(gh sub-issue create \
    --parent {epic_number} \
    --title "{task_name_from_frontmatter}" \
    --body-file .claude/epics/$ARGUMENTS/{task_file} \
    --label "task,epic:$ARGUMENTS" \
    --json number -q .number)
else
  # Fallback: Create regular issue
  task_number=$(gh issue create \
    --title "{task_name_from_frontmatter}" \
    --body-file .claude/epics/$ARGUMENTS/{task_file} \
    --label "task,epic:$ARGUMENTS" \
    --json number -q .number)
fi
```

Store each returned issue number.

### 3. Rename Task Files

After creating each task, rename the file from numbered to issue ID:
```bash
# Original: 001.md, 002.md, etc.
# New: 1234.md, 1235.md, etc.

for task_file in .claude/epics/$ARGUMENTS/[0-9][0-9][0-9].md; do
  # Get the issue number we just created for this task
  task_number={stored_from_creation}
  
  # Rename file
  mv "$task_file" ".claude/epics/$ARGUMENTS/${task_number}.md"
  
  # Update github field in frontmatter
  # Already done during creation, but ensure it's set
done
```

### 4. Update Epic with Task List (Fallback Only)

If NOT using gh-sub-issue, add task list to epic:

```bash
if [ "$use_subissues" = false ]; then
  # Get current epic body
  gh issue view {epic_number} --json body -q .body > /tmp/epic-body.md
  
  # Append task list
  cat >> /tmp/epic-body.md << 'EOF'
  
  ## Tasks
  - [ ] #{task1_number} {task1_name}
  - [ ] #{task2_number} {task2_name}
  - [ ] #{task3_number} {task3_name}
  EOF
  
  # Update epic issue
  gh issue edit {epic_number} --body-file /tmp/epic-body.md
fi
```

With gh-sub-issue, this is automatic!

### 5. Update Local Files

Follow `/rules/frontmatter-operations.md` to update:
- Epic file: Add `github: {issue_url}` to frontmatter
- Each task file (now named by issue ID): Add `github: {issue_url}` to frontmatter
- Update `updated` field with current datetime (see `/rules/datetime.md`)

### 6. Create Mapping File

Create `.claude/epics/$ARGUMENTS/github-mapping.md`:
```markdown
# GitHub Issue Mapping

Epic: #{number} - {url}

Tasks:
- #{number}: {title} - {url}
- #{number}: {title} - {url}

Synced: {current_datetime}
```

### 6. Create Worktree

Follow `/rules/worktree-operations.md` to create development worktree:

```bash
# Ensure main is current
git checkout main
git pull origin main

# Create worktree for epic
git worktree add ../epic-$ARGUMENTS -b epic/$ARGUMENTS

echo "✅ Created worktree: ../epic-$ARGUMENTS"
```

### 7. Output

```
✅ Synced to GitHub
  - Epic: #{number} (with task list)
  - Tasks: {count} issues created
  - Progress tracking: Enabled on epic
  - Labels: epic:$ARGUMENTS
  - Worktree: ../epic-$ARGUMENTS

Next steps:
  - Start parallel execution: /pm:epic-start $ARGUMENTS
  - Or work on single issue: /pm:issue-start {task_number}
  - View progress: https://github.com/{owner}/{repo}/issues/{epic_number}
```

## Error Handling

Follow `/rules/github-operations.md` for GitHub CLI errors.

If any issue creation fails:
- Report what succeeded
- Note what failed
- Don't attempt rollback (partial sync is fine)

## Important Notes

- Trust GitHub CLI authentication
- Don't pre-check for duplicates
- Update frontmatter only after successful creation
- Keep operations simple and atomic