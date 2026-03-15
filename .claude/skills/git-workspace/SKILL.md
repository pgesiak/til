---
name: git-workspace
description: Clone a private GitHub repo (via HTTPS + PAT), create a working branch, use all repo files as context, and persist every artifact (new or updated) back to the repo with automatic git commit + push. Use this skill whenever the user provides a GitHub repo URL with a PAT token, asks to "work in a repo", "save artifacts to git", "use my repo as workspace", wants persistent git-tracked conversation artifacts, or mentions syncing work to a remote repository. Also trigger when the user says "git workspace", "repo workspace", "clone and work", or wants to pick up where they left off from a git-backed session.
---

# Git Workspace

Turns a private GitHub repo into a persistent, git-tracked workspace for conversation artifacts. Every file you create or update is committed and pushed to a dedicated branch — giving you a full audit trail of the conversation's output.

## When This Skill Triggers

The user provides two inputs (explicitly or via env vars):
1. **Repo URL** — GitHub HTTPS URL (e.g., `https://github.com/user/repo.git`)
2. **PAT** — GitHub Personal Access Token with `repo` scope

Example trigger phrases:
- "Here's my repo https://github.com/user/til.git and PAT ghp_xxx, let's work"
- "Clone my repo and save all artifacts there"
- "Use this repo as our workspace: ..."
- "Set up git workspace for https://..."

## Setup Procedure

Execute these steps **immediately** when the skill triggers. Do not ask for confirmation — just do it.

### Step 1: Extract Credentials

Parse the repo URL and PAT from the user's message. Accept these formats:

```
# Inline in message
https://github.com/user/repo.git PAT: ghp_xxxx

# Or as env vars (check first)
GIT_WORKSPACE_REPO=https://github.com/user/repo.git
GIT_WORKSPACE_PAT=ghp_xxxx

# Or two separate values in the message
repo: https://github.com/user/repo
pat: ghp_xxxx
```

### Step 2: Clone the Repo

```bash
# Construct authenticated URL
# Format: https://<PAT>@github.com/user/repo.git
REPO_URL="https://${PAT}@github.com/${OWNER}/${REPO}.git"

# Clone to /home/claude/workspace/<repo-name>
WORKSPACE="/home/claude/workspace/${REPO_NAME}"
git clone --depth=50 "${REPO_URL}" "${WORKSPACE}"
cd "${WORKSPACE}"

# Configure git identity
git config user.email "claude@workspace.local"
git config user.name "Claude Workspace"
```

### Step 3: Create Working Branch

Always create a new branch. Never commit directly to `main` or `master`.

```bash
# Generate branch name: claude/<date>-<short-session-id>
BRANCH="claude/$(date +%Y%m%d-%H%M%S)"
git checkout -b "${BRANCH}"
```

Report to user: "Workspace ready. Working on branch `claude/YYYYMMDD-HHMMSS` in `/home/claude/workspace/<repo>`."

### Step 4: Index Repo Contents

Build a mental map of the repo so you can reference existing files:

```bash
# Show repo structure (respect .gitignore)
find . -not -path './.git/*' -type f | head -200

# Read key files for context
cat README.md 2>/dev/null
cat .claude/* 2>/dev/null
```

Read files that are relevant to the user's task. Don't read everything — read on demand as the conversation requires it.

## Artifact Lifecycle

Every artifact you produce follows this cycle: **create/update → commit → push**.

### Creating a New Artifact

```bash
# 1. Write the file to the repo workspace
cat > "${WORKSPACE}/path/to/artifact.md" << 'EOF'
<content>
EOF

# 2. Stage and commit with a descriptive message
cd "${WORKSPACE}"
git add "path/to/artifact.md"
git commit -m "add: path/to/artifact.md - <brief description>"

# 3. Push to origin
git push origin "${BRANCH}"
```

### Updating an Existing Artifact

```bash
# 1. Edit the file (use edit tool or write tool)
# 2. Stage, commit with what changed
cd "${WORKSPACE}"
git add "path/to/artifact.md"
git commit -m "update: path/to/artifact.md - <what changed>"

# 3. Push
git push origin "${BRANCH}"
```

### Batch Operations

When creating or updating multiple files in a single logical operation, batch them into one commit:

```bash
git add file1.md file2.md file3.md
git commit -m "add: analysis artifacts - <description>"
git push origin "${BRANCH}"
```

## Commit Message Convention

Use this format:
```
<action>: <path> - <description>

Actions:
  add:    new file
  update: modified file
  remove: deleted file
  rename: moved/renamed file
  batch:  multiple files in one logical change
```

## Reading Repo Files

When the user references repo content or you need context:

```bash
# Read specific file
cat "${WORKSPACE}/path/to/file"

# Search for content
grep -r "pattern" "${WORKSPACE}" --include="*.md" -l

# Find files by name
find "${WORKSPACE}" -name "*.py" -not -path '*/.git/*'
```

Always read from the workspace path, not from search indexes.

## Pull Before Major Operations

If the session is long-running, pull before creating new artifacts to avoid conflicts:

```bash
cd "${WORKSPACE}"
git pull origin "${BRANCH}" --rebase 2>/dev/null || true
```

## Error Handling

- **Clone fails**: Check PAT permissions (`repo` scope required). Report the exact error.
- **Push fails**: Pull + rebase first. If conflict, report to user.
- **PAT expired**: Ask user for a new PAT, re-configure remote: `git remote set-url origin "https://${NEW_PAT}@github.com/..."`

## Session Resume

If the user returns and wants to continue on an existing branch:

```bash
cd "${WORKSPACE}"
git fetch origin
git checkout claude/<existing-branch>
git pull origin claude/<existing-branch>
```

## Security Notes

- The PAT is used only for git operations in this session
- Never echo the PAT in output, logs, or committed files
- Add the PAT to the authenticated remote URL only (not to files)
- If the repo has a `.env` or secrets, do not read or display them unless explicitly asked

## Workspace Layout Convention

Organize artifacts in the repo under a consistent path:

```
<repo>/
├── README.md                    # Existing repo content
├── .claude/                     # Claude workspace metadata
│   └── skills/                  # Skills (if applicable)
└── artifacts/                   # Default artifact output directory
    └── YYYY-MM-DD/              # Date-grouped if many artifacts
        ├── analysis.md
        ├── architecture.md
        └── ...
```

If the user specifies a different path, use that instead. Adapt to the repo's existing structure.

## Checklist (execute on every trigger)

1. [ ] Parse repo URL + PAT from user input or env vars
2. [ ] Clone repo to `/home/claude/workspace/<name>`
3. [ ] Configure git identity
4. [ ] Create branch `claude/<timestamp>`
5. [ ] Push branch to origin (establishes remote tracking)
6. [ ] Index repo structure, read relevant files
7. [ ] Report workspace status to user
8. [ ] For every artifact: write → commit → push
9. [ ] Never leak PAT in output or committed files
