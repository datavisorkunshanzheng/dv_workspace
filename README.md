# DV Workspace

A multi-repo development workspace for DataVisor projects, enabling unified codebase management and parallel feature development through git worktrees.

## Why DV Workspace?

**The Problem**: Working on multiple tickets across multiple repos is painful:
- Switching branches loses uncommitted work or requires stashing
- Hard to context-switch between urgent hotfix and ongoing feature work
- Each repo tracks its own branch independently - easy to lose sync

**The Solution**: DV Workspace combines two powerful git features:
1. **Submodules** - All repos in one place, tracked together
2. **Worktrees** - Isolated directories for each task, no branch switching needed

```
Before (traditional workflow):
  feature-platform/  ← constantly switching branches, stashing changes
  qa-test/           ← hoping branches stay in sync

After (DV Workspace):
  dv_workspace/
  ├── feature-platform/     ← stays on release branch (clean)
  ├── qa-test/              ← stays on release branch (clean)
  ├── WT__PI-100/           ← hotfix work (isolated)
  │   ├── feature-platform/
  │   └── qa-test/
  └── WT__FEATURE-456/      ← feature work (isolated)
      └── feature-platform/
```

## Initial Setup

```bash
# Clone with all submodules
git clone --recurse-submodules git@github.com:datavisorcode/dv_workspace.git

# Or if already cloned without submodules
git submodule update --init --recursive
```

## Repository Structure

```
dv_workspace/
├── CLAUDE.md              # AI agent context and architecture docs
├── README.md              # This file
├── worktree               # Worktree management script
├── .gitmodules            # Submodule definitions
│
├── feature-platform/      # Java 8 backend (submodule)
├── internal-ui/           # Java 17 backend (submodule)
├── ngsc/                  # Node.js bridge (submodule)
├── qa-test/               # E2E tests (submodule)
├── charts/                # Helm values (submodule)
├── infra/                 # Infrastructure (submodule)
│
└── WT__<session>/         # Worktree sessions (gitignored)
    └── <repo>/            # Isolated working directory
```

## How It Works

### Concept Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│  dv_workspace (parent repo)                                     │
│                                                                 │
│  ┌──────────────────┐  ┌──────────────────┐                     │
│  │ feature-platform │  │ qa-test          │  ← Submodules       │
│  │ (main repo)      │  │ (main repo)      │    on release       │
│  │                  │  │                  │    branch           │
│  │ DV.202602A.Ext   │  │ DV.202602A.Ext   │                     │
│  └────────┬─────────┘  └────────┬─────────┘                     │
│           │                     │                               │
│           │ git worktree add    │                               │
│           ▼                     ▼                               │
│  ┌─────────────────────────────────────────┐                    │
│  │ WT__PI-100/                             │  ← Session         │
│  │  ├── feature-platform/                  │    directory       │
│  │  │   (branch: DV.202602A.Ext.PI-100)    │                    │
│  │  └── qa-test/                           │                    │
│  │      (branch: DV.202602A.Ext.PI-100)    │                    │
│  └─────────────────────────────────────────┘                    │
│                                                                 │
│  ┌───────────────────────────────────────────┐                  │
│  │ WT__FEATURE-789/                          │  ← Another       │
│  │  └── feature-platform/                    │    session       │
│  │      (branch: DV.202602A.Ext.FEATURE-789) │                  │
│  └───────────────────────────────────────────┘                  │
└─────────────────────────────────────────────────────────────────┘
```

### Branch Naming Convention

When you create a session, branches are named: `<base-branch>.<session-name>`

| Base Branch | Session | Resulting Branch |
|-------------|---------|------------------|
| `DV.202602A.External` | `PI-100` | `DV.202602A.External.PI-100` |
| `master` | `hotfix-123` | `master.hotfix-123` |

## Using the Worktree Script

### Commands

| Command | Description |
|---------|-------------|
| `./worktree new -b <base> <session> <repos...>` | Create a new session |
| `./worktree add <session> <repos...>` | Add repos to existing session |
| `./worktree remove <session> <repos...>` | Remove repos from session |
| `./worktree list [session]` | List sessions or repos in a session |
| `./worktree clean <session> [--delete-branches]` | Remove a session |

### Quick Reference

```bash
# Create a new session for ticket PI-100 based on release branch
./worktree new -b DV.202602A.External PI-100 feature-platform qa-test

# Add another repo to the session
./worktree add PI-100 internal-ui

# List all active sessions
./worktree list
# Output:
#   PI-100
#   FEATURE-789

# List repos in a specific session
./worktree list PI-100
# Output:
#   feature-platform/
#     Branch: DV.202602A.External.PI-100
#     Path:   /path/to/dv_workspace/WT__PI-100/feature-platform/
#   qa-test/
#     Branch: DV.202602A.External.PI-100
#     Path:   /path/to/dv_workspace/WT__PI-100/qa-test/

# Remove a repo from session (keeps branch by default)
./worktree remove PI-100 qa-test

# Clean up entire session (keeps branches)
./worktree clean PI-100

# Clean up and delete branches
./worktree clean PI-100 --delete-branches
```

## Typical Workflow

### 1. Starting a New Task

```bash
# Create worktrees for the repos you need to modify
./worktree new -b DV.202602A.External PI-100 feature-platform qa-test

# Navigate to the session and start working
cd WT__PI-100/feature-platform

# You're now on branch: DV.202602A.External.PI-100
git status
```

### 2. Context Switching

```bash
# Urgent hotfix comes in while working on PI-100

# Just create another session - no stashing, no branch switching
./worktree new -b DV.202602A.External HOTFIX-999 feature-platform

# Work on hotfix
cd WT__HOTFIX-999/feature-platform
# ... fix the bug ...
git commit -m "Fix critical bug"
git push origin DV.202602A.External.HOTFIX-999

# Switch back to PI-100 - your work is exactly where you left it
cd ../../WT__PI-100/feature-platform
```

### 3. Working with AI Coding Assistants

```bash
# Start Claude Code in a session directory for focused work
cd WT__PI-100
claude

# Claude will see feature-platform/ and qa-test/ in context
# and understand it's working on the PI-100 task
```

### 4. Completing a Task

```bash
# Push changes and create PR
cd WT__PI-100/feature-platform
git push origin DV.202602A.External.PI-100
# Create PR via GitHub...

# After PR is merged, clean up
cd ../..
./worktree clean PI-100 --delete-branches
```

### 5. Syncing with Upstream

```bash
# Update your worktree with latest changes from base branch
cd WT__PI-100/feature-platform
git fetch origin
git rebase origin/DV.202602A.External
```

## Best Practices

1. **Name sessions after tickets**: Use ticket IDs (e.g., `PI-100`, `DEVBUG-19444`) for easy tracking

2. **Include only needed repos**: Don't add all repos to every session - only the ones you'll modify

3. **Clean up completed sessions**: Run `./worktree clean` after merging to avoid clutter

4. **Keep main repos clean**: Do all feature work in worktree sessions; main directories stay on release branches

5. **Commit submodule updates**: When you finish work, update the parent repo's submodule pointers:
   ```bash
   cd /path/to/dv_workspace
   git add feature-platform
   git commit -m "Update feature-platform submodule pointer"
   ```

## Troubleshooting

### "Worktree already exists"
The worktree directory already exists. Either clean it up or use a different session name:
```bash
./worktree clean <session-name>
# or
./worktree new -b <branch> <different-session-name> <repos...>
```

### "Base branch not found"
The specified base branch doesn't exist. Fetch from remote first:
```bash
cd feature-platform
git fetch origin
git branch -a | grep <branch-name>  # verify it exists
```

### Stale worktree references
If git complains about existing worktrees that don't exist:
```bash
cd feature-platform
git worktree prune
```

### Worktree has uncommitted changes
The `clean` command will fail if there are uncommitted changes. Either commit, stash, or use `--force`:
```bash
cd WT__<session>/<repo>
git stash  # or git commit
cd ../..
./worktree clean <session>
```

## See Also

- `CLAUDE.md` - Technical architecture documentation for AI agents
- `./worktree --help` - Full command documentation
- `./worktree <command> --help` - Command-specific help
