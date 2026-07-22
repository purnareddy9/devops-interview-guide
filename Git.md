---
layout: default
title: "🔀 Git — DevOps Interview Guide"
render_with_liquid: false
---

# 🔀 Git — DevOps Interview Guide

> [← Networking](./Networking.md) | [Main Index](./README.md) | [Docker →](./Docker.md)

---

## Table of Contents

1. [Git Internals](#git-internals)
2. [Core Operations](#core-operations)
3. [Branching & Merging](#branching--merging)
4. [Rebasing](#rebasing)
5. [Git Workflows](#git-workflows)
6. [Git Hooks](#git-hooks)
7. [Advanced Operations](#advanced-operations)
8. [Interview Questions](#interview-questions)
9. [Scenario-Based Questions](#scenario-based-questions)
10. [Hands-On Labs](#hands-on-labs)
11. [Cheat Sheet](#cheat-sheet)

---

## Git Internals

### How Git Stores Data

```
Working Directory  →  Staging Area (Index)  →  Repository (.git/)
                   git add                  git commit

.git/
├── HEAD           → Points to current branch ref
├── config         → Repository configuration
├── objects/       → All content (blobs, trees, commits, tags)
│   ├── ab/        → First 2 chars of SHA-1 hash
│   │   └── cd1234...  → Object file
│   ├── pack/      → Packed objects (compressed)
├── refs/
│   ├── heads/     → Branch references
│   └── tags/      → Tag references
└── COMMIT_EDITMSG → Last commit message
```

### Git Object Types

| Object | Description | Content |
|--------|-------------|---------|
| **Blob** | File content | Raw file data |
| **Tree** | Directory structure | List of blobs + trees with names |
| **Commit** | Snapshot pointer | Tree hash, parent(s), author, message |
| **Tag** | Named ref | Points to a commit with metadata |

```bash
# Inspect git objects
git cat-file -t abc1234    # Type of object
git cat-file -p abc1234    # Content of object
git log --oneline          # Short commit hashes
git show HEAD:path/to/file # File content at HEAD
```

---

## Core Operations

### Essential Commands

```bash
# Initialize
git init                          # New repo
git clone https://github.com/... # Clone remote

# Status & inspection
git status                        # Working tree status
git diff                          # Unstaged changes
git diff --staged                 # Staged vs last commit
git log --oneline --graph --all   # Visual branch history
git log --author="Alice" --since="2 weeks ago"
git blame file.py                 # Who changed each line

# Staging
git add file.py                   # Stage specific file
git add -p                        # Interactive hunk staging
git add .                         # Stage all changes
git reset HEAD file.py            # Unstage (keep changes)

# Committing
git commit -m "feat: add user auth"
git commit --amend                # Modify last commit
git commit --amend --no-edit      # Add to last commit silently

# Remote operations
git remote add origin https://...
git fetch origin                  # Download, don't merge
git pull origin main              # fetch + merge
git pull --rebase origin main     # fetch + rebase
git push origin feature-branch
git push --force-with-lease       # Safe force push

# Undoing changes
git restore file.py               # Discard working dir changes
git restore --staged file.py      # Unstage
git revert abc1234                # Create new commit undoing
git reset --soft HEAD~1           # Undo commit, keep staged
git reset --mixed HEAD~1          # Undo commit, keep unstaged
git reset --hard HEAD~1           # Undo commit, discard changes
```

---

## Branching & Merging

### Branch Operations

```bash
# Create & switch
git branch feature/login          # Create branch
git checkout feature/login        # Switch to branch
git checkout -b feature/login     # Create + switch (classic)
git switch -c feature/login       # Create + switch (modern)

# List branches
git branch -a                     # All (local + remote)
git branch -v                     # With last commit
git branch --merged               # Branches merged into current
git branch --no-merged            # Unmerged branches

# Delete
git branch -d feature/login       # Delete (safe)
git branch -D feature/login       # Force delete
git push origin --delete feature/login  # Delete remote

# Rename
git branch -m old-name new-name
```

### Merge Strategies

```bash
# Fast-forward merge (clean, linear history)
git merge feature/login
# If possible, just moves pointer forward

# No-fast-forward (preserves merge commit)
git merge --no-ff feature/login

# Squash merge (all commits → one commit)
git merge --squash feature/login
git commit -m "feat: add login feature"

# Merge with message
git merge feature/login -m "Merge feature/login into main"

# Abort a merge conflict
git merge --abort

# Resolving conflicts
git status                        # See conflicted files
# Edit files to resolve <<<< ==== >>>> markers
git add resolved_file.py
git commit                        # Complete the merge
```

### Conflict Markers

```python
<<<<<<< HEAD
def authenticate(user, password):
    return check_ldap(user, password)  # Our version
=======
def authenticate(user, password):
    return check_database(user, password)  # Their version
>>>>>>> feature/login
```

---

## Rebasing

### Rebase vs Merge

```
Before:
  A - B - C  (main)
       \
        D - E  (feature)

After merge:
  A - B - C - M  (main, M = merge commit)
       \       /
        D - E

After rebase:
  A - B - C - D' - E'  (feature rebased onto main)
```

```bash
# Rebase feature onto main
git checkout feature/login
git rebase main

# Interactive rebase (last 4 commits)
git rebase -i HEAD~4
# Commands: pick, reword, edit, squash, fixup, drop

# Squash commits during rebase
# Change 'pick' to 'squash' for commits to combine

# Rebase onto specific branch
git rebase origin/main

# Continue after resolving conflict
git rebase --continue

# Abort rebase
git rebase --abort
```

### Interactive Rebase Example

```
pick abc123 feat: add login form
squash def456 fix: login form styles
squash ghi789 fix: typo in login
reword jkl012 refactor: extract auth logic
drop mno345 WIP: debugging
```

---

## Git Workflows

### GitFlow

```
main         ──────●────────────────────────●──
                   │                        │
develop      ──────●────●────●────●─────────●──
                        │    │    │
feature           ──────●    │    │
                             │    │
release                 ─────●    │
                                  │
hotfix                       ─────●
```

```bash
# GitFlow commands
git flow init
git flow feature start user-auth
git flow feature finish user-auth
git flow release start 1.2.0
git flow release finish 1.2.0
git flow hotfix start critical-bug
git flow hotfix finish critical-bug
```

### Trunk-Based Development

```
main  ─────●─────●─────●─────●─────●────>
           │     │     │     │
        short  short  short  short
        lived  lived  lived  lived
```

- Feature flags to hide incomplete features
- Short-lived branches (< 1 day ideally)
- CI runs on every commit to main
- Continuous deployment

### GitHub Flow (simplified)

```
1. Create branch from main
2. Add commits
3. Open Pull Request
4. Review & discuss
5. Deploy to test environment
6. Merge to main
7. Deploy main to production
```

---

## Git Hooks

### Common Hooks

| Hook | Trigger | Use Case |
|------|---------|---------|
| `pre-commit` | Before commit created | Lint, format, unit tests |
| `commit-msg` | After commit message written | Enforce message format |
| `pre-push` | Before push to remote | Run full test suite |
| `post-merge` | After successful merge | Update dependencies |
| `pre-receive` | Server-side: before push accepted | Policy enforcement |
| `post-receive` | Server-side: after push accepted | Trigger CI/CD, notify |

### Pre-commit Hook Example

```bash
#!/bin/bash
# .git/hooks/pre-commit (chmod +x)

# Run linter
echo "Running flake8..."
if ! flake8 .; then
    echo "Linting failed! Commit aborted."
    exit 1
fi

# Check for debug statements
if grep -rn "debugger\|console.log\|pdb.set_trace" --include="*.py" --include="*.js" .; then
    echo "Found debug statements! Remove before committing."
    exit 1
fi

# Run tests
echo "Running tests..."
if ! python -m pytest tests/ -q; then
    echo "Tests failed! Commit aborted."
    exit 1
fi

echo "All checks passed!"
```

### Commit Message Hook

```bash
#!/bin/bash
# .git/hooks/commit-msg

COMMIT_MSG_FILE="$1"
COMMIT_MSG=$(cat "$COMMIT_MSG_FILE")

# Enforce Conventional Commits format
PATTERN="^(feat|fix|docs|style|refactor|test|chore|ci)(\(.+\))?: .{1,100}$"

if ! echo "$COMMIT_MSG" | grep -qE "$PATTERN"; then
    echo "Invalid commit message format!"
    echo "Expected: type(scope): description"
    echo "Types: feat, fix, docs, style, refactor, test, chore, ci"
    echo "Example: feat(auth): add JWT token validation"
    exit 1
fi
```

---

## Advanced Operations

### Cherry-Pick

```bash
# Apply a specific commit to current branch
git cherry-pick abc1234

# Cherry-pick a range
git cherry-pick abc1234..def5678

# Cherry-pick without committing
git cherry-pick -n abc1234

# Cherry-pick with merge commit
git cherry-pick -m 1 abc1234
```

### Stash

```bash
git stash                         # Stash changes
git stash push -m "WIP: login"    # Named stash
git stash list                    # List stashes
git stash apply stash@{1}         # Apply specific stash
git stash pop                     # Apply + remove top stash
git stash drop stash@{1}          # Delete specific stash
git stash branch feature/branch   # Create branch from stash
```

### Tags

```bash
git tag v1.2.0                    # Lightweight tag
git tag -a v1.2.0 -m "Release 1.2.0"  # Annotated tag
git push origin v1.2.0            # Push tag
git push origin --tags            # Push all tags
git tag -d v1.2.0                 # Delete local tag
git push origin --delete v1.2.0  # Delete remote tag
```

### Submodules

```bash
git submodule add https://github.com/... path/to/sub
git submodule init
git submodule update
git submodule update --init --recursive  # Deep init
git submodule foreach git pull origin main
```

### Bisect (Bug Finding)

```bash
git bisect start
git bisect bad                    # Current commit is broken
git bisect good v1.0.0            # This commit was good
# Git checks out middle commit; test it
git bisect bad                    # Or: git bisect good
# Repeat until first bad commit found
git bisect reset                  # Exit bisect
```

---

## Interview Questions

### 🟢 Beginner

**Q1: What is the difference between `git pull` and `git fetch`?**

> `git fetch` downloads changes from remote but does NOT merge them — your working branch is unchanged. You can review changes with `git diff origin/main`. `git pull` = `git fetch` + `git merge` (or `git rebase` with `--rebase` flag). Best practice: use `git fetch` then review, then merge manually.

**Q2: Explain `git rebase` vs `git merge`.**

> Both integrate changes from one branch to another.
> - **Merge** preserves history as-is, creates a merge commit, non-destructive
> - **Rebase** rewrites commit history by replaying commits on top of another branch, creates cleaner linear history
> - **Rule**: Never rebase public/shared branches (rewrites history others depend on). Rebase private feature branches before merging.

**Q3: What does `git reset --hard` do?**

> Moves the current branch pointer AND updates the staging area AND working directory to match the specified commit. **All uncommitted changes are lost.** Dangerous: you cannot recover with Git unless you know the previous commit SHA (check `git reflog` within ~30 days).

**Q4: How do you undo a commit that has already been pushed?**

> Use `git revert <commit-sha>` — this creates a new commit that undoes the changes, preserving history. This is safe for shared branches. **Never** use `git reset --hard` on public branches as it rewrites history and will break other developers' local repos.

---

### 🟡 Intermediate

**Q5: Explain GitFlow and when you'd use it vs Trunk-Based Development.**

> **GitFlow** has `main`, `develop`, `feature/*`, `release/*`, `hotfix/*` branches. Good for:
> - Scheduled release cycles (e.g., ship every 2 weeks)
> - Multiple versions in production simultaneously
> - Teams that need explicit release preparation
>
> **Trunk-Based Development** — everyone commits to `main` frequently with short-lived branches.Good for:
> - Continuous deployment pipelines
> - Teams practicing CI/CD properly
> - Modern microservice architectures

**Q6: What is a `detached HEAD` state?**

> Normally HEAD points to a branch name (which points to a commit). In detached HEAD, HEAD points directly to a commit SHA — not a branch. Any commits you make here are "orphaned" and not reachable from any branch (garbage collected eventually). Happens when: `git checkout <commit-sha>`, `git checkout v1.0 (tag)`. Fix: `git checkout main` or create a branch: `git checkout -b my-branch`.

**Q7: How would you find a commit that introduced a bug?**

> Use `git bisect` — binary search through commit history:
> ```bash
> git bisect start
> git bisect bad HEAD           # Current is broken
> git bisect good v1.2.0        # This version was fine
> # Git checks out middle commit
> # Test the code
> git bisect good/bad           # Tell git the result
> # Repeat ~ log2(n) times
> git bisect reset
> ```
> Can also automate: `git bisect run pytest tests/test_feature.py`

---

### 🔴 Advanced

**Q8: Explain how `git reflog` saves you from disaster.**

> `git reflog` is a local log of all HEAD movements — every checkout, commit, reset, merge. It's the "undo history" of your Git operations. If you accidentally `git reset --hard HEAD~5`, use `git reflog` to find the SHA before the reset, then `git reset --hard <sha>` to recover. The reflog is local-only and expires after ~90 days.

**Q9: How does Git handle merge conflicts at a low level?**

> Git uses a **3-way merge**: it compares the two branch tips (ours/theirs) with their **merge base** (common ancestor). If both sides changed the same region from the base, it's a conflict. Git marks conflicting sections with `<<<<<<<`, `=======`, `>>>>>>>` markers. Tools: `git mergetool` (vimdiff, kdiff3), or `git checkout --ours/--theirs file` to choose one side entirely.

**Q10: What is `git worktree` and when is it useful?**

> `git worktree` allows multiple working directories from the same repo, each checked out at a different branch. Useful when:
> - You need to fix a hotfix while mid-feature without stashing
> - Running tests on multiple branches simultaneously
> ```bash
> git worktree add ../hotfix-dir hotfix/critical
> git worktree list
> git worktree remove ../hotfix-dir
> ```

---

## Scenario-Based Questions

### 🔵 Scenario 1: Accidentally Committed Secrets

*"A developer pushed an API key to the main branch. What do you do?"*

```bash
# IMMEDIATE: Revoke the API key (most important!)
# Then remove from history:

# 1. Use git filter-repo (modern, preferred)
pip install git-filter-repo
git filter-repo --path-glob '*.env' --invert-paths

# 2. Or BFG Repo Cleaner (easier)
java -jar bfg.jar --replace-text secrets.txt my-repo.git

# 3. Force push all branches
git push origin --force --all
git push origin --force --tags

# 4. All collaborators must re-clone or:
git fetch --all
git reset --hard origin/main

# 5. Add to .gitignore immediately
echo ".env" >> .gitignore
echo "*.pem" >> .gitignore

# 6. Set up git-secrets or truffleHog pre-commit hook
pip install detect-secrets
detect-secrets scan > .secrets.baseline
```

### 🔵 Scenario 2: Merge Conflict in CI Pipeline

*"Your CI pipeline fails on merge conflicts. How do you set up a process to prevent this?"*

```bash
# 1. Enforce branch freshness
# Add to PR requirements: branch must be up-to-date with main

# 2. Rebase workflow
git fetch origin
git rebase origin/main
# Resolve conflicts locally, then push

# 3. Automated merge conflict detection
# In CI:
git merge --no-commit --no-ff origin/main
if [ $? -ne 0 ]; then
    echo "Merge conflict detected"
    git merge --abort
    exit 1
fi
git merge --abort  # Don't actually merge

# 4. Use GitHub/GitLab "Merge Queue" feature
# Serializes merges, ensures each PR merges cleanly
```

---

## Hands-On Labs

### Lab 1: Interactive Rebase Practice

```bash
# Create a test repo with messy history
mkdir git-lab && cd git-lab
git init
for i in 1 2 3 4 5; do
    echo "Content $i" > file$i.txt
    git add .
    git commit -m "WIP commit $i"
done

# Now clean up with interactive rebase
git log --oneline

# Squash all into one meaningful commit
git rebase -i HEAD~5
# Change all but first 'pick' to 'squash'
# Write a good commit message
git log --oneline  # Should be one clean commit
```

### Lab 2: Bisect Automation

```bash
# Simulate a regression
git init bisect-lab && cd bisect-lab
for i in $(seq 1 20); do
    echo "feature $i" > app.py
    [[ $i -eq 14 ]] && echo "BUG_INTRODUCED=1" >> app.py
    git add . && git commit -m "commit $i"
done

# Bisect with a test script
git bisect start
git bisect bad HEAD
git bisect good HEAD~19
git bisect run bash -c 'grep -q BUG_INTRODUCED app.py && exit 1 || exit 0'
```

---

## Cheat Sheet

```bash
# ─── SETUP ───────────────────────────────────────────
git config --global user.name  "Your Name"
git config --global user.email "you@example.com"
git config --global core.editor "vim"
git config --global pull.rebase true

# ─── SNAPSHOT ────────────────────────────────────────
git status -s            # Short status
git diff HEAD            # All changes since last commit
git log --oneline -10    # Last 10 commits

# ─── BRANCH SHORTCUTS ────────────────────────────────
git switch -c feature    # New branch
git switch main          # Switch to main
git branch -D feature    # Force delete

# ─── COMMON ALIASES ──────────────────────────────────
git config --global alias.st   status
git config --global alias.lg   "log --oneline --graph --all"
git config --global alias.undo "reset --soft HEAD~1"
git config --global alias.discard "restore ."

# ─── COLLABORATION ───────────────────────────────────
git fetch --prune           # Fetch + remove stale remote refs
git remote -v               # List remotes
git remote set-url origin https://new-url

# ─── EMERGENCY RECOVERY ──────────────────────────────
git reflog                  # All recent HEAD movements
git fsck --lost-found       # Find dangling objects
git stash list              # List all stashes
```

---

> **Cross-links:** [← Networking](./Networking.md) | [Docker →](./Docker.md) | [CI/CD (Git-based workflows) →](./CICD.md) | [Jenkins (uses Git) →](./Jenkins.md)