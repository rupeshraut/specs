# Effective Modern Git Usage

A comprehensive guide to mastering Git workflows, best practices, advanced techniques, and team collaboration strategies.

---

## Table of Contents

1. [Git Fundamentals](#git-fundamentals)
2. [Modern Git Workflows](#modern-git-workflows)
3. [Branch Management](#branch-management)
4. [Commit Best Practices](#commit-best-practices)
5. [Advanced Git Commands](#advanced-git-commands)
6. [Rebasing and History Rewriting](#rebasing-and-history-rewriting)
7. [Merge Strategies](#merge-strategies)
8. [Git Hooks and Automation](#git-hooks-and-automation)
9. [Resolving Conflicts](#resolving-conflicts)
10. [Git Stash and Working Directory](#git-stash-and-working-directory)
11. [Searching and Debugging](#searching-and-debugging)
12. [Collaboration Workflows](#collaboration-workflows)
13. [Git Configuration and Aliases](#git-configuration-and-aliases)
14. [Monorepo Strategies](#monorepo-strategies)
15. [Security and Signing](#security-and-signing)
16. [Best Practices](#best-practices)

---

## Git Fundamentals

### Git Object Model

```
Git stores data as snapshots, not differences:

┌─────────────────────────────────────┐
│ Commit                              │
│ ├─ Tree (directory snapshot)        │
│ │  ├─ Blob (file content)          │
│ │  ├─ Blob (file content)          │
│ │  └─ Tree (subdirectory)          │
│ ├─ Parent commit(s)                 │
│ └─ Metadata (author, date, message) │
└─────────────────────────────────────┘

Three States:
├─ Working Directory (modified)
├─ Staging Area (staged)
└─ Repository (committed)
```

### Essential Configuration

```bash
# Global configuration
git config --global user.name "Your Name"
git config --global user.email "your.email@example.com"

# Default branch name
git config --global init.defaultBranch main

# Editor
git config --global core.editor "vim"
git config --global core.editor "code --wait"  # VS Code

# Line endings (important for cross-platform teams)
git config --global core.autocrlf input    # Mac/Linux
git config --global core.autocrlf true     # Windows

# Colors
git config --global color.ui auto

# Default pull behavior (recommended)
git config --global pull.rebase true

# Prune on fetch
git config --global fetch.prune true

# Reuse recorded resolution (rerere)
git config --global rerere.enabled true

# View configuration
git config --list
git config --list --show-origin  # Show where each config is set
```

---

## Modern Git Workflows

### 1. **GitHub Flow** (Simple, Continuous Deployment)

```
Production-Ready Workflow:

main (always deployable)
  │
  ├─ feature/user-authentication
  │  ├─ commit: Add login form
  │  ├─ commit: Add password validation
  │  └─ Pull Request → Review → Merge
  │
  ├─ feature/shopping-cart
  │  ├─ commit: Add cart model
  │  └─ Pull Request → Review → Merge
  │
  └─ hotfix/security-patch
     └─ Pull Request → Review → Merge

Rules:
1. Main branch is always deployable
2. Create descriptive branches from main
3. Push to branch regularly
4. Open Pull Request when ready
5. Merge after review and CI passes
6. Deploy immediately after merge
```

**Implementation:**

```bash
# Start new feature
git checkout main
git pull origin main
git checkout -b feature/user-authentication

# Make changes and commit
git add .
git commit -m "Add login form with validation"

# Push and create PR
git push -u origin feature/user-authentication
# Create Pull Request on GitHub

# After PR approved, merge (on GitHub)
# Then locally:
git checkout main
git pull origin main
git branch -d feature/user-authentication
```

### 2. **Git Flow** (Complex, Release-Based)

```
Release-Based Workflow:

main (production releases only)
  │
develop (integration branch)
  │
  ├─ feature/new-feature
  │  └─ merge → develop
  │
  ├─ release/1.2.0
  │  ├─ bug fixes only
  │  ├─ merge → main (tag v1.2.0)
  │  └─ merge → develop
  │
  └─ hotfix/critical-bug
     ├─ merge → main (tag v1.1.1)
     └─ merge → develop

Branch Types:
├─ main: Production releases
├─ develop: Integration branch
├─ feature/*: New features
├─ release/*: Release preparation
└─ hotfix/*: Production fixes
```

**Implementation:**

```bash
# Initialize Git Flow
git flow init

# Start new feature
git flow feature start user-authentication
# Work on feature
git add .
git commit -m "Add authentication"
git flow feature finish user-authentication

# Start release
git flow release start 1.2.0
# Fix release bugs
git commit -m "Fix release bugs"
git flow release finish 1.2.0

# Hotfix
git flow hotfix start security-patch
git commit -m "Fix security issue"
git flow hotfix finish security-patch
```

### 3. **Trunk-Based Development** (Modern, CI/CD Friendly)

```
Continuous Integration Workflow:

main (trunk)
  │
  ├─ Short-lived feature branch (< 2 days)
  │  └─ merge → main (multiple times per day)
  │
  ├─ Another short-lived branch
  │  └─ merge → main
  │
  └─ release/v1.2.0 (branch from main)
     └─ cherry-pick fixes if needed

Principles:
1. Main branch is always green (tests pass)
2. Small, frequent commits to main
3. Feature flags for incomplete features
4. Fast feedback from CI/CD
5. Release branches for production
```

**Implementation:**

```bash
# Quick feature (< 2 days)
git checkout -b quick-feature
# Make small changes
git commit -m "Add feature flag for new UI"
git push origin quick-feature

# Immediate review and merge
# Back to main
git checkout main
git pull origin main

# Release from main
git checkout -b release/1.2.0
git tag -a v1.2.0 -m "Release 1.2.0"
git push origin release/1.2.0 --tags
```

---

## Branch Management

### Creating and Switching Branches

```bash
# Create and switch (old way)
git checkout -b feature/new-feature

# Create and switch (new way, Git 2.23+)
git switch -c feature/new-feature

# Switch to existing branch
git switch main
git switch -  # Switch to previous branch

# Create branch without switching
git branch feature/new-feature

# Create branch from specific commit
git branch feature/new-feature abc123

# Create branch from remote
git checkout -b feature/new-feature origin/feature/new-feature
git switch -c feature/new-feature origin/feature/new-feature
```

### Branch Operations

```bash
# List branches
git branch                 # Local branches
git branch -r              # Remote branches
git branch -a              # All branches
git branch -v              # With last commit
git branch -vv             # With tracking info

# Rename branch
git branch -m old-name new-name
git branch -m new-name     # Rename current branch

# Delete branch
git branch -d feature/done        # Safe delete (merged only)
git branch -D feature/abandoned   # Force delete

# Delete remote branch
git push origin --delete feature/done
git push origin :feature/done     # Old syntax

# Track remote branch
git branch -u origin/main
git branch --set-upstream-to=origin/main

# View merged/unmerged branches
git branch --merged        # Merged to current branch
git branch --no-merged     # Not merged yet
git branch --merged main   # Merged to main
```

### Branch Cleanup

```bash
# Delete all merged branches
git branch --merged main | grep -v "main" | xargs git branch -d

# Prune remote-tracking branches
git fetch --prune
git fetch -p

# Remove branches that don't exist on remote
git remote prune origin

# Interactive branch cleanup
git for-each-ref --format='%(refname:short)' refs/heads | \
  grep -v "main\|develop" | \
  xargs -I {} sh -c 'echo "Delete {}?" && read && git branch -d {}'
```

---

## Commit Best Practices

### Conventional Commits

```
Format: <type>(<scope>): <subject>

Types:
├─ feat: New feature
├─ fix: Bug fix
├─ docs: Documentation
├─ style: Formatting, missing semicolons, etc.
├─ refactor: Code restructuring
├─ perf: Performance improvement
├─ test: Adding tests
├─ chore: Build process, dependencies
├─ ci: CI/CD changes
└─ revert: Revert previous commit

Examples:
feat(auth): add JWT token authentication
fix(api): resolve null pointer in user endpoint
docs(readme): update installation instructions
refactor(database): optimize query performance
perf(cache): implement Redis caching layer
test(user): add unit tests for user service
chore(deps): upgrade Spring Boot to 3.2.0
ci(github): add automated deployment workflow
```

### Atomic Commits

```bash
# ✅ GOOD: One logical change per commit
git add src/auth/login.java
git commit -m "feat(auth): add login form validation"

git add src/auth/logout.java
git commit -m "feat(auth): implement logout functionality"

# ❌ BAD: Multiple unrelated changes
git add .
git commit -m "Fixed stuff and added features"

# Interactive staging for atomic commits
git add -p  # Stage in chunks
# y = stage this hunk
# n = don't stage
# s = split into smaller hunks
# e = manually edit hunk
```

### Writing Good Commit Messages

```bash
# Template
<type>(<scope>): <subject>

<body>

<footer>

# Example
feat(user-profile): add profile picture upload

- Implement multipart file upload endpoint
- Add image validation (size, format)
- Integrate with S3 storage
- Add error handling for upload failures

Closes #123
Refs #456

# Set commit template
git config --global commit.template ~/.gitmessage
```

### Amending Commits

```bash
# Amend last commit message
git commit --amend -m "New message"

# Amend last commit with new changes
git add forgotten-file.txt
git commit --amend --no-edit

# Change author of last commit
git commit --amend --author="Name <email@example.com>"

# Amend without changing timestamp
git commit --amend --no-edit --date="$(date)"

# ⚠️ WARNING: Never amend pushed commits (unless force-pushing to feature branch)
```

---

## Advanced Git Commands

### Interactive Rebase

```bash
# Rebase last 5 commits
git rebase -i HEAD~5

# Rebase from specific commit
git rebase -i abc123

# Common operations in interactive rebase:
# pick   = use commit
# reword = change commit message
# edit   = stop to amend commit
# squash = combine with previous commit
# fixup  = like squash, discard message
# drop   = remove commit

# Example:
pick abc123 feat: add login
reword def456 fix: validation bug  # Will prompt for new message
squash ghi789 fix: typo            # Combine with previous
fixup jkl012 fix: another typo     # Combine, discard message
drop mno345 wip: temporary changes # Remove this commit
```

### Cherry-Pick

```bash
# Apply specific commit
git cherry-pick abc123

# Cherry-pick multiple commits
git cherry-pick abc123 def456 ghi789

# Cherry-pick range (exclusive)
git cherry-pick abc123..def456

# Cherry-pick without committing
git cherry-pick -n abc123
git cherry-pick --no-commit abc123

# Cherry-pick with message edit
git cherry-pick -e abc123

# Continue after resolving conflicts
git cherry-pick --continue

# Abort cherry-pick
git cherry-pick --abort
```

### Reflog - Recovering Lost Commits

```bash
# View reflog
git reflog
git reflog show HEAD

# Reflog for specific branch
git reflog show main

# Recover deleted commit
git reflog
# Find commit hash (e.g., abc123)
git checkout abc123
git branch recovered-branch

# Recover deleted branch
git reflog | grep "branch-name"
git checkout -b branch-name abc123

# Undo reset
git reset --hard HEAD@{1}

# Time-based reflog
git reflog show HEAD@{2.days.ago}
git diff HEAD@{0} HEAD@{1.day.ago}
```

### Bisect - Finding Bugs

```bash
# Start bisect
git bisect start

# Mark current as bad
git bisect bad

# Mark last known good commit
git bisect good abc123

# Git will checkout middle commit
# Test and mark as good or bad
git bisect good   # If commit is good
git bisect bad    # If commit has bug

# Continue until bug is found
# Git will tell you the first bad commit

# Automate bisect with script
git bisect run ./test-script.sh

# Reset when done
git bisect reset

# Example automated bisect
git bisect start HEAD v1.0
git bisect run mvn test
```

### Worktree - Multiple Working Directories

```bash
# Create new worktree
git worktree add ../project-feature feature/new-feature

# Create worktree for new branch
git worktree add ../project-hotfix -b hotfix/urgent-fix

# List worktrees
git worktree list

# Remove worktree
git worktree remove ../project-feature

# Prune stale worktrees
git worktree prune

# Use case: Work on multiple branches simultaneously
git worktree add ../project-feature feature/new-feature
git worktree add ../project-review feature/code-review
# Now you can have both directories open in different editors
```

---

## Rebasing and History Rewriting

### Interactive Rebase Scenarios

```bash
# Squash last 3 commits
git rebase -i HEAD~3
# Mark last 2 as "squash"

# Reorder commits
git rebase -i HEAD~5
# Reorder the pick lines

# Split a commit
git rebase -i HEAD~3
# Mark commit as "edit"
git reset HEAD^
git add file1.txt
git commit -m "First part"
git add file2.txt
git commit -m "Second part"
git rebase --continue

# Remove sensitive data from history
git rebase -i HEAD~5
# Mark commit as "drop"

# Fixup commits
git commit --fixup abc123
git rebase -i --autosquash HEAD~5
```

### Rebase vs Merge

```bash
# Merge (preserves history)
git checkout main
git merge feature/new-feature
# Creates merge commit

# Rebase (linear history)
git checkout feature/new-feature
git rebase main
git checkout main
git merge feature/new-feature  # Fast-forward merge

# When to use:
# Merge:
#   - Public branches (main, develop)
#   - Preserve collaboration history
#   - Team prefers merge commits
#
# Rebase:
#   - Feature branches before merging
#   - Keep linear history
#   - Clean up local commits
```

### Handling Rebase Conflicts

```bash
# Start rebase
git rebase main

# If conflicts occur:
# 1. Fix conflicts in files
# 2. Stage resolved files
git add resolved-file.txt

# Continue rebase
git rebase --continue

# Skip current commit
git rebase --skip

# Abort rebase
git rebase --abort

# Rebase with strategy
git rebase -X theirs main     # Prefer incoming changes
git rebase -X ours main       # Prefer current changes
```

---

## Merge Strategies

### Fast-Forward Merge

```bash
# Fast-forward (default if possible)
git merge feature/new-feature

# Force fast-forward (fail if not possible)
git merge --ff-only feature/new-feature

# Prevent fast-forward (always create merge commit)
git merge --no-ff feature/new-feature
```

### Merge Strategies

```bash
# Recursive (default)
git merge feature/new-feature

# Ours (keep our changes in conflicts)
git merge -X ours feature/new-feature

# Theirs (keep their changes in conflicts)
git merge -X theirs feature/new-feature

# Octopus (multiple branches)
git merge feature1 feature2 feature3

# Squash merge (combine all commits into one)
git merge --squash feature/new-feature
git commit -m "feat: add new feature"
```

### Merge Commit Messages

```bash
# Custom merge commit message
git merge --no-ff -m "Merge feature: user authentication" feature/auth

# Edit merge commit message
git merge --no-ff -e feature/auth

# Merge without commit (review first)
git merge --no-commit feature/auth
# Review changes
git commit -m "Custom merge message"
```

---

## Git Hooks and Automation

### Client-Side Hooks

```bash
# Hooks directory
cd .git/hooks

# Common hooks (remove .sample extension to activate)
# pre-commit - Run before commit
# prepare-commit-msg - Prepare commit message
# commit-msg - Validate commit message
# post-commit - After commit
# pre-push - Before push
# pre-rebase - Before rebase
```

### Pre-Commit Hook Example

```bash
# .git/hooks/pre-commit
#!/bin/bash

# Run tests
echo "Running tests..."
npm test
if [ $? -ne 0 ]; then
    echo "Tests failed. Commit aborted."
    exit 1
fi

# Run linter
echo "Running linter..."
npm run lint
if [ $? -ne 0 ]; then
    echo "Linting failed. Commit aborted."
    exit 1
fi

# Check for debug statements
if git diff --cached | grep -E "(console\\.log|debugger|TODO|FIXME)"; then
    echo "Debug statements found. Please remove before committing."
    exit 1
fi

echo "Pre-commit checks passed!"
```

### Commit Message Hook

```bash
# .git/hooks/commit-msg
#!/bin/bash

commit_msg_file=$1
commit_msg=$(cat "$commit_msg_file")

# Validate conventional commit format
if ! echo "$commit_msg" | grep -qE "^(feat|fix|docs|style|refactor|perf|test|chore|ci)(\(.+\))?: .{10,}"; then
    echo "❌ Invalid commit message format!"
    echo "Format: <type>(<scope>): <subject>"
    echo "Example: feat(auth): add login functionality"
    exit 1
fi

# Check minimum message length
if [ ${#commit_msg} -lt 10 ]; then
    echo "❌ Commit message too short (minimum 10 characters)"
    exit 1
fi

echo "✅ Commit message valid"
```

### Pre-Push Hook

```bash
# .git/hooks/pre-push
#!/bin/bash

# Prevent push to main
branch=$(git rev-parse --abbrev-ref HEAD)
if [ "$branch" = "main" ]; then
    echo "❌ Direct push to main is not allowed!"
    echo "Please create a pull request instead."
    exit 1
fi

# Run integration tests
echo "Running integration tests..."
mvn verify
if [ $? -ne 0 ]; then
    echo "❌ Integration tests failed. Push aborted."
    exit 1
fi

echo "✅ Pre-push checks passed"
```

### Husky (Git Hooks Manager)

```bash
# Install Husky
npm install --save-dev husky

# Initialize
npx husky install

# Add hooks
npx husky add .husky/pre-commit "npm test"
npx husky add .husky/commit-msg "npx commitlint --edit $1"
npx husky add .husky/pre-push "npm run build"
```

---

## Resolving Conflicts

### Understanding Conflicts

```bash
# When conflicts occur
git merge feature/new-feature
# Auto-merging file.txt
# CONFLICT (content): Merge conflict in file.txt

# Conflict markers in file:
<<<<<<< HEAD
Current branch changes
=======
Incoming changes
>>>>>>> feature/new-feature
```

### Resolving Conflicts

```bash
# View conflicted files
git status
git diff

# Manual resolution:
# 1. Edit files to resolve conflicts
# 2. Remove conflict markers
# 3. Keep desired changes

# After resolving
git add resolved-file.txt
git commit  # Or git merge --continue

# Use mergetool
git mergetool

# Accept all theirs/ours
git checkout --theirs file.txt
git checkout --ours file.txt

# Show conflict in different style
git config merge.conflictstyle diff3
# Shows:
# <<<<<<< HEAD
# Current
# ||||||| base
# Original
# =======
# Incoming
# >>>>>>> branch
```

### Diff and Merge Tools

```bash
# Configure diff tool
git config --global diff.tool vimdiff
git config --global difftool.prompt false

# Configure merge tool
git config --global merge.tool meld
git config --global mergetool.prompt false

# Use tools
git difftool
git mergetool

# Popular tools:
# - VS Code: code --wait --diff
# - Meld: meld
# - KDiff3: kdiff3
# - P4Merge: p4merge
```

---

## Git Stash and Working Directory

### Stashing Changes

```bash
# Stash current changes
git stash
git stash save "WIP: working on feature"

# Stash with untracked files
git stash -u
git stash --include-untracked

# Stash with message and keep index
git stash save --keep-index "Message"

# List stashes
git stash list

# Apply stash
git stash apply            # Apply most recent
git stash apply stash@{2}  # Apply specific stash

# Apply and remove
git stash pop
git stash pop stash@{2}

# Drop stash
git stash drop stash@{2}

# Clear all stashes
git stash clear

# Create branch from stash
git stash branch new-branch-name stash@{0}
```

### Stash Advanced

```bash
# Partial stash (interactive)
git stash -p
# Same as git add -p

# Show stash contents
git stash show
git stash show -p stash@{0}  # Show as diff

# Apply specific files from stash
git checkout stash@{0} -- file.txt
```

---

## Searching and Debugging

### Git Grep

```bash
# Search in tracked files
git grep "search term"

# Search with line numbers
git grep -n "function name"

# Search in specific commit
git grep "search" abc123

# Count matches
git grep -c "TODO"

# Search in specific files
git grep "pattern" -- "*.java"

# Show function name
git grep -p "search term"

# Case insensitive
git grep -i "search"
```

### Git Log Advanced

```bash
# Compact log
git log --oneline
git log --oneline -10  # Last 10 commits

# Graph view
git log --graph --oneline --all

# Pretty format
git log --pretty=format:"%h %an %ar %s"
# %h = hash, %an = author, %ar = date, %s = subject

# Show files changed
git log --stat
git log --name-only
git log --name-status

# Search commits
git log --grep="bug fix"
git log --author="John"
git log --since="2 weeks ago"
git log --until="2024-01-01"

# Show commits that touched file
git log -- path/to/file.txt

# Show commits that added/removed string
git log -S "function name"
git log -G "regex pattern"

# Show merge commits only
git log --merges

# Show non-merge commits
git log --no-merges

# Diff between branches
git log main..feature/branch
git log feature/branch --not main
```

### Blame and History

```bash
# See who changed each line
git blame file.txt

# Ignore whitespace changes
git blame -w file.txt

# Show email
git blame -e file.txt

# Show line range
git blame -L 10,20 file.txt

# See commit that introduced line
git blame -L 10,+5 file.txt
git show <commit-hash>

# Follow file renames
git log --follow --stat -- file.txt

# Show changes to function
git log -L :functionName:file.txt
```

---

## Collaboration Workflows

### Pull Request Workflow

```bash
# 1. Create feature branch
git checkout -b feature/new-feature

# 2. Make changes and commit
git add .
git commit -m "feat: add new feature"

# 3. Push to remote
git push -u origin feature/new-feature

# 4. Create Pull Request on GitHub/GitLab

# 5. Address review comments
git add .
git commit -m "fix: address review comments"
git push

# 6. After approval, squash commits (optional)
git rebase -i HEAD~3

# 7. Merge PR (on GitHub/GitLab)

# 8. Cleanup
git checkout main
git pull origin main
git branch -d feature/new-feature
```

### Code Review Best Practices

```bash
# Fetch PR for local review
git fetch origin pull/123/head:pr-123
git checkout pr-123

# Or add to .git/config
[remote "origin"]
    fetch = +refs/pull/*/head:refs/remotes/origin/pr/*

# Then
git fetch origin
git checkout pr-123

# Review changes
git diff main...pr-123
git log main..pr-123

# Comment on specific lines (use GitHub/GitLab UI)

# Approve/Request changes (use GitHub/GitLab UI)
```

### Keeping Fork Up-to-Date

```bash
# Add upstream remote (one-time)
git remote add upstream https://github.com/original/repo.git

# Fetch upstream
git fetch upstream

# Merge upstream main
git checkout main
git merge upstream/main

# Or rebase
git rebase upstream/main

# Push to your fork
git push origin main

# Update feature branch with latest main
git checkout feature/my-feature
git rebase main
git push --force-with-lease origin feature/my-feature
```

---

## Git Configuration and Aliases

### Useful Aliases

```bash
# Add to ~/.gitconfig
[alias]
    # Status
    st = status -sb
    
    # Commits
    co = checkout
    br = branch
    ci = commit
    cm = commit -m
    ca = commit --amend
    
    # Logs
    lg = log --graph --pretty=format:'%Cred%h%Creset -%C(yellow)%d%Creset %s %Cgreen(%cr) %C(bold blue)<%an>%Creset' --abbrev-commit
    ls = log --pretty=format:"%C(yellow)%h%Cred%d\\ %Creset%s%Cblue\\ [%cn]" --decorate
    ll = log --pretty=format:"%C(yellow)%h%Cred%d\\ %Creset%s%Cblue\\ [%cn]" --decorate --numstat
    
    # Diff
    df = diff
    dc = diff --cached
    dlc = diff --cached HEAD^
    
    # Stash
    sl = stash list
    sa = stash apply
    ss = stash save
    
    # Undo
    undo = reset --soft HEAD^
    unstage = reset HEAD --
    
    # Branch management
    branches = branch -a
    remotes = remote -v
    tags = tag -l
    
    # Cleanup
    cleanup = "!git branch --merged | grep -v '\\*\\|main\\|develop' | xargs -n 1 git branch -d"
    
    # Quick add and commit
    ac = "!git add -A && git commit -m"
    acp = "!git add -A && git commit -m $1 && git push"
    
    # Amend last commit
    amend = commit --amend --no-edit
    
    # Find file
    find = "!git ls-files | grep -i"
    
    # Contributors
    contributors = shortlog --summary --numbered
```

### Advanced Configuration

```bash
# Credential helper
git config --global credential.helper cache
git config --global credential.helper 'cache --timeout=3600'

# Diff algorithm
git config --global diff.algorithm histogram

# Push default
git config --global push.default current
git config --global push.autoSetupRemote true

# Rebase by default on pull
git config --global pull.rebase true

# Auto-correct commands
git config --global help.autocorrect 20  # Wait 2 seconds

# Ignore file permissions
git config --global core.fileMode false

# Signing commits
git config --global commit.gpgsign true
git config --global user.signingkey <GPG-KEY-ID>

# Large file storage (LFS)
git lfs install
git lfs track "*.psd"
git add .gitattributes
```

---

## Monorepo Strategies

### Sparse Checkout

```bash
# Clone without files
git clone --no-checkout <repo-url>
cd <repo>

# Enable sparse checkout
git sparse-checkout init --cone

# Add directories to checkout
git sparse-checkout set folder1 folder2

# List sparse checkout
git sparse-checkout list

# Disable sparse checkout
git sparse-checkout disable
```

### Submodules

```bash
# Add submodule
git submodule add https://github.com/example/lib.git libs/example

# Clone repo with submodules
git clone --recursive <repo-url>

# Initialize submodules in existing repo
git submodule init
git submodule update

# Or combined
git submodule update --init --recursive

# Update submodules
git submodule update --remote

# Remove submodule
git submodule deinit libs/example
git rm libs/example
rm -rf .git/modules/libs/example
```

### Git Subtree

```bash
# Add subtree
git subtree add --prefix=libs/example https://github.com/example/lib.git main --squash

# Update subtree
git subtree pull --prefix=libs/example https://github.com/example/lib.git main --squash

# Push changes to subtree
git subtree push --prefix=libs/example https://github.com/example/lib.git main
```

---

## Security and Signing

### GPG Signing

```bash
# Generate GPG key
gpg --gen-key

# List keys
gpg --list-secret-keys --keyid-format=long

# Configure Git to use key
git config --global user.signingkey <KEY-ID>
git config --global commit.gpgsign true

# Sign specific commit
git commit -S -m "Signed commit"

# Verify signatures
git log --show-signature
git verify-commit <commit-hash>

# Export public key for GitHub
gpg --armor --export <KEY-ID>
```

### Protecting Sensitive Data

```bash
# Never commit secrets!
# Use .gitignore
echo "*.env" >> .gitignore
echo "secrets.yml" >> .gitignore
echo "*.key" >> .gitignore

# Remove from history (nuclear option)
git filter-branch --force --index-filter \
  "git rm --cached --ignore-unmatch path/to/secret" \
  --prune-empty --tag-name-filter cat -- --all

# Or use BFG Repo-Cleaner (faster)
bfg --delete-files secret.key
bfg --replace-text passwords.txt
git reflog expire --expire=now --all && git gc --prune=now --aggressive

# ⚠️ WARNING: This rewrites history. Coordinate with team!
```

---

## Best Practices

### ✅ DO

1. **Commit often, push regularly**
   ```bash
   git commit -m "feat: add validation" && git push
   ```

2. **Write meaningful commit messages**
   ```bash
   # Good
   git commit -m "fix(auth): resolve token expiration bug in logout flow"
   
   # Bad
   git commit -m "fixed stuff"
   ```

3. **Use branches for features**
   ```bash
   git checkout -b feature/user-profile
   ```

4. **Pull before push**
   ```bash
   git pull --rebase origin main
   git push
   ```

5. **Review before committing**
   ```bash
   git diff
   git status
   git add -p  # Review each change
   ```

6. **Use .gitignore**
   ```bash
   # Common ignores
   node_modules/
   .env
   *.log
   .DS_Store
   target/
   *.class
   ```

7. **Keep commits atomic**
   ```bash
   git add specific-file.txt
   git commit -m "feat: specific change"
   ```

8. **Sync regularly**
   ```bash
   git fetch --prune
   git pull origin main
   ```

9. **Use tags for releases**
   ```bash
   git tag -a v1.0.0 -m "Release 1.0.0"
   git push origin v1.0.0
   ```

10. **Clean up merged branches**
    ```bash
    git branch --merged | grep -v "main" | xargs git branch -d
    ```

### ❌ DON'T

1. **Don't commit large files** - Use Git LFS
2. **Don't commit secrets** - Use environment variables
3. **Don't rewrite public history** - Only rebase local branches
4. **Don't work directly on main** - Use feature branches
5. **Don't force push to shared branches**
   ```bash
   # Only force push to your feature branches
   git push --force-with-lease origin feature/my-branch
   ```
6. **Don't commit commented code** - Delete it (it's in history)
7. **Don't commit generated files** - Add to .gitignore
8. **Don't mix formatting and logic changes** - Separate commits
9. **Don't commit without reviewing** - Always use git diff
10. **Don't use git add .** blindly - Review what you're staging

---

## Troubleshooting Common Issues

### Undo Common Mistakes

```bash
# Undo last commit (keep changes)
git reset --soft HEAD^

# Undo last commit (discard changes)
git reset --hard HEAD^

# Undo changes to file
git checkout -- file.txt
git restore file.txt

# Undo staged changes
git reset HEAD file.txt
git restore --staged file.txt

# Undo pushed commit (create new commit)
git revert HEAD
git push

# Fix wrong branch
git reset --hard origin/correct-branch

# Recover deleted branch
git reflog
git checkout -b recovered-branch <commit-hash>

# Fix detached HEAD
git switch -c new-branch  # Create branch from current state
# or
git switch main  # Go back to main
```

### Performance Issues

```bash
# Reduce repo size
git gc --aggressive --prune=now

# Remove unnecessary files
git clean -fd  # Remove untracked files and directories
git clean -fX  # Remove ignored files

# Shallow clone (faster)
git clone --depth 1 <repo-url>

# Partial clone (Git 2.19+)
git clone --filter=blob:none <repo-url>
```

---

## Conclusion

**Key Takeaways:**

1. **Workflow**: Choose workflow that fits your team (GitHub Flow, Git Flow, Trunk-Based)
2. **Commits**: Atomic, meaningful, conventional format
3. **Branches**: Feature branches, descriptive names, clean up merged branches
4. **Collaboration**: Pull requests, code reviews, keep fork updated
5. **History**: Linear (rebase) vs merge commits (team decision)
6. **Tools**: Hooks, aliases, GUI tools for productivity
7. **Security**: Sign commits, never commit secrets

**Remember**: Git is a tool to help you and your team. Use it effectively, but don't over-complicate. Start simple, add complexity as needed!
