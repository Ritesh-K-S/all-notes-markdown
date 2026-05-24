# Git — Complete Notes

---

## Table of Contents

1. [Introduction](#1-introduction)
2. [Configuration](#2-configuration)
3. [Repository Basics](#3-repository-basics)
4. [Basic Workflow](#4-basic-workflow)
5. [Viewing History](#5-viewing-history)
6. [Branching](#6-branching)
7. [Merging](#7-merging)
8. [Rebasing](#8-rebasing)
9. [Stashing](#9-stashing)
10. [Undoing Changes](#10-undoing-changes)
11. [Remote Repositories](#11-remote-repositories)
12. [Tagging](#12-tagging)
13. [Cherry-Pick](#13-cherry-pick)
14. [Advanced Commands](#14-advanced-commands)
15. [Git Hooks](#15-git-hooks)
16. [Git Workflows](#16-git-workflows)
17. [SSH Setup](#17-ssh-setup)
18. [.gitattributes](#18-gitattributes)
19. [Best Practices](#19-best-practices)
20. [Quick Reference Cheat Sheet](#20-quick-reference-cheat-sheet)

---

## 1. INTRODUCTION

### What is Git?
- **Distributed Version Control System (DVCS)** — tracks changes in source code.
- Created by **Linus Torvalds** (2005) for Linux kernel development.
- Every developer has the **full repository** history locally.
- Fast, efficient, supports **branching & merging** natively.
- Industry standard for source code management.

### Git vs Other VCS
| Feature | Git (Distributed) | SVN (Centralized) |
|---------|-------------------|-------------------|
| Repository | Full copy on every machine | Single central server |
| Offline work | Full functionality | Limited |
| Branching | Fast & lightweight | Slow & heavy |
| Speed | Very fast (local operations) | Slower (network dependent) |
| Storage | Snapshots | Differences (deltas) |

### How Git Works
```
Working Directory  →  Staging Area (Index)  →  Local Repository  →  Remote Repository
     (edit)            (git add)                (git commit)         (git push)
```

### Three States of Files
| State | Description |
|-------|-------------|
| **Modified** | Changed but not staged |
| **Staged** | Marked to go into next commit |
| **Committed** | Safely stored in local database |

### Install Git
```bash
# Linux
sudo apt install git          # Debian/Ubuntu
sudo yum install git          # RHEL/CentOS
sudo pacman -S git            # Arch

# macOS
brew install git
# or install Xcode Command Line Tools

# Windows
# Download from https://git-scm.com/downloads

# Verify
git --version
```

---

## 2. CONFIGURATION

### Config Levels
| Level | Flag | Location | Scope |
|-------|------|----------|-------|
| System | `--system` | `/etc/gitconfig` | All users on machine |
| Global | `--global` | `~/.gitconfig` | Current user (all repos) |
| Local | `--local` | `.git/config` | Current repository only |

Priority: **local > global > system**

### Essential Setup
```bash
# Identity (required)
git config --global user.name "Ritesh Singh"
git config --global user.email "ritesh@example.com"

# Default branch name
git config --global init.defaultBranch main

# Default editor
git config --global core.editor "code --wait"     # VS Code
git config --global core.editor "vim"
git config --global core.editor "nano"
git config --global core.editor "notepad"

# Line endings
git config --global core.autocrlf true     # Windows (CRLF → LF on commit)
git config --global core.autocrlf input    # macOS/Linux (CRLF → LF)

# Color output
git config --global color.ui auto

# Default merge tool
git config --global merge.tool vscode
git config --global mergetool.vscode.cmd 'code --wait $MERGED'

# Default diff tool
git config --global diff.tool vscode
git config --global difftool.vscode.cmd 'code --wait --diff $LOCAL $REMOTE'

# Pull strategy
git config --global pull.rebase true       # rebase on pull (cleaner history)
git config --global pull.ff only           # fast-forward only

# Push default
git config --global push.default current   # push current branch to same name
git config --global push.autoSetupRemote true   # auto set upstream (Git 2.37+)

# Aliases
git config --global alias.st status
git config --global alias.co checkout
git config --global alias.br branch
git config --global alias.ci commit
git config --global alias.lg "log --oneline --graph --all --decorate"
git config --global alias.last "log -1 HEAD"
git config --global alias.unstage "reset HEAD --"
```

### View Configuration
```bash
git config --list                     # all settings
git config --list --show-origin       # settings + file location
git config --global --list            # global settings only
git config user.name                  # specific setting
git config --global --edit            # open config in editor
```

### Credential Storage
```bash
git config --global credential.helper cache                # cache 15 min
git config --global credential.helper 'cache --timeout=3600'  # cache 1 hour
git config --global credential.helper store                # store in plain text (insecure)
git config --global credential.helper manager              # OS credential manager (Windows)
git config --global credential.helper osxkeychain          # macOS Keychain
```

---

## 3. REPOSITORY BASICS

### Create Repository
```bash
# Initialize new repo in current directory
git init

# Initialize with specific branch name
git init -b main

# Initialize in a new directory
git init my-project

# Clone existing repository
git clone https://github.com/user/repo.git
git clone https://github.com/user/repo.git my-folder    # custom folder name
git clone --depth 1 https://github.com/user/repo.git    # shallow clone (latest only)
git clone --branch dev https://github.com/user/repo.git  # specific branch
git clone --recurse-submodules https://github.com/user/repo.git  # with submodules

# Clone via SSH
git clone git@github.com:user/repo.git
```

### .gitignore
```bash
# .gitignore — files/folders Git should ignore

# Compiled files
*.class
*.jar
*.war
*.ear
*.o
*.pyc
__pycache__/

# Build output
target/
build/
dist/
out/
bin/

# IDE files
.idea/
*.iml
.vscode/
*.swp
*.swo
.project
.classpath
.settings/

# OS files
.DS_Store
Thumbs.db
desktop.ini

# Environment / secrets
.env
.env.local
*.pem
*.key

# Dependencies
node_modules/
vendor/

# Logs
*.log
logs/

# Patterns
*.tmp                 # all .tmp files
!important.tmp        # except this one
docs/*.txt            # .txt in docs/ only
docs/**/*.txt         # .txt in docs/ and subdirectories
/TODO                 # only root TODO (not subdir/TODO)
build/                # directory (trailing slash)
```

### .gitignore Global
```bash
# Create global ignore
git config --global core.excludesfile ~/.gitignore_global

# ~/.gitignore_global
.DS_Store
Thumbs.db
*.swp
.idea/
.vscode/
```

### Track Already Ignored Files
```bash
# Force add an ignored file
git add -f secret.yml

# Stop tracking a file (but keep locally)
git rm --cached file.txt
git rm --cached -r folder/

# Check what's being ignored
git status --ignored
git check-ignore -v filename
```

---

## 4. BASIC WORKFLOW

### Status
```bash
git status                # full status
git status -s             # short format
git status -sb            # short + branch info
git status --ignored      # show ignored files
```

### Short Status Symbols
```
 M file.txt       # modified (not staged)
M  file.txt       # modified (staged)
MM file.txt       # modified (staged + more unstaged changes)
A  file.txt       # new file (staged)
?? file.txt       # untracked
D  file.txt       # deleted
R  file.txt       # renamed
!! file.txt       # ignored (with --ignored)
```

### Add (Stage)
```bash
git add file.txt              # stage specific file
git add file1.txt file2.txt   # stage multiple files
git add *.java                # stage by pattern
git add src/                  # stage entire directory
git add .                     # stage all changes (new + modified + deleted)
git add -A                    # stage all (same as .)
git add -u                    # stage modified + deleted only (not new files)
git add -p                    # interactive staging (patch mode)
                              # y=stage, n=skip, s=split, e=edit, q=quit
git add -N file.txt           # mark intent to add (track but don't stage content)
```

### Commit
```bash
git commit                           # open editor for message
git commit -m "Add login feature"    # inline message
git commit -m "Title" -m "Body"      # title + body
git commit -am "msg"                 # add tracked files + commit (skip staging)
git commit --amend                   # modify last commit (message + content)
git commit --amend -m "New message"  # change last commit message
git commit --amend --no-edit         # add staged changes to last commit silently
git commit --allow-empty -m "msg"    # commit with no changes (CI triggers)
git commit --date="2026-01-01 12:00:00" -m "msg"  # custom date
```

### Commit Message Convention
```
type(scope): subject          ← header (≤50 chars)
                               ← blank line
body (optional)                ← wrap at 72 chars
                               ← blank line
footer (optional)              ← breaking changes, issue refs

Types:
  feat     — new feature
  fix      — bug fix
  docs     — documentation only
  style    — formatting (no code change)
  refactor — code change (no feature/fix)
  test     — adding/updating tests
  chore    — build, CI, dependencies
  perf     — performance improvement
  ci       — CI/CD changes
  build    — build system changes
  revert   — revert previous commit

Examples:
  feat(auth): add JWT token authentication
  fix(api): handle null response from payment gateway
  docs: update README with setup instructions
  refactor(user): extract validation to service layer
  chore(deps): upgrade Spring Boot to 3.2.0

Footer:
  BREAKING CHANGE: API endpoint changed from /v1 to /v2
  Closes #123
  Fixes #456
  Refs #789
```

### Remove Files
```bash
git rm file.txt                   # delete + stage deletion
git rm --cached file.txt          # untrack (keep file locally)
git rm -r folder/                 # remove directory
git rm --cached -r folder/        # untrack directory
git rm '*.log'                    # remove by pattern
```

### Move / Rename Files
```bash
git mv old-name.txt new-name.txt
git mv file.txt subfolder/
# Equivalent to: mv old new && git rm old && git add new
```

---

## 5. VIEWING HISTORY

### Log
```bash
git log                              # full log
git log --oneline                    # compact (hash + message)
git log --oneline --graph            # with branch graph
git log --oneline --graph --all      # all branches
git log --oneline -10                # last 10 commits
git log --stat                       # file change stats
git log --shortstat                  # summary stats only
git log -p                           # show diffs (patches)
git log -p file.txt                  # diffs for specific file
git log -- file.txt                  # history of specific file
git log --follow file.txt            # follow renames
git log --author="Ritesh"            # by author
git log --grep="fix"                 # search commit messages
git log --since="2026-01-01"         # after date
git log --until="2026-03-01"         # before date
git log --since="2 weeks ago"        # relative date
git log --after="2026-01-01" --before="2026-03-01"
git log --no-merges                  # exclude merge commits
git log --merges                     # only merge commits
git log --diff-filter=D              # only deleted files
git log --diff-filter=A              # only added files
git log --name-only                  # show changed filenames
git log --name-status                # filenames + status (A/M/D)
git log main..feature               # commits in feature not in main
git log feature --not main           # same as above
git log --all --source               # show which ref reached each commit
git log --pretty=format:"%h %an %s"  # custom format
```

### Log Format Specifiers
| Specifier | Description |
|-----------|-------------|
| `%H` | Full commit hash |
| `%h` | Short commit hash |
| `%an` | Author name |
| `%ae` | Author email |
| `%ad` | Author date |
| `%ar` | Author date (relative) |
| `%cn` | Committer name |
| `%s` | Subject (first line) |
| `%b` | Body |
| `%d` | Decorations (branches, tags) |
| `%n` | Newline |

```bash
# Pretty formats
git log --pretty=format:"%h - %an, %ar : %s"
git log --pretty=format:"%C(yellow)%h%C(reset) %C(green)%an%C(reset) %s %C(blue)(%ar)%C(reset)"

# Useful aliases
git log --oneline --graph --all --decorate    # visual branch tree
git log --format='%h %ad %s' --date=short     # compact with date
```

### Show Specific Commit
```bash
git show abc1234                     # show commit details + diff
git show HEAD                        # latest commit
git show HEAD~2                      # 2 commits ago
git show HEAD:file.txt               # file content at HEAD
git show abc1234:src/App.java        # file at specific commit
git show --stat abc1234              # just file stats
git show --name-only abc1234         # just filenames
```

### Diff
```bash
git diff                             # unstaged changes (working vs staging)
git diff --staged                    # staged changes (staging vs last commit)
git diff --cached                    # same as --staged
git diff HEAD                        # all changes (working vs last commit)
git diff abc1234 def5678             # between two commits
git diff main feature                # between two branches
git diff main...feature              # changes in feature since diverging from main
git diff file.txt                    # diff for specific file
git diff --stat                      # summary only
git diff --name-only                 # changed filenames only
git diff --name-status               # filenames + status
git diff --word-diff                 # inline word-level diff
git diff --color-words               # colored word-level diff
git diff HEAD~3..HEAD                # last 3 commits
git diff --check                     # check for whitespace errors
```

### Blame (Who changed what)
```bash
git blame file.txt                   # show line-by-line author
git blame -L 10,20 file.txt         # specific line range
git blame -L 10,+5 file.txt         # 5 lines from line 10
git blame -w file.txt               # ignore whitespace changes
git blame --date=short file.txt     # compact date format
git blame -e file.txt               # show email instead of name
```

### Shortlog
```bash
git shortlog                         # commits grouped by author
git shortlog -sn                     # commit count by author (sorted)
git shortlog -sn --no-merges         # exclude merges
git shortlog --since="2026-01-01"    # within date range
```

---

## 6. BRANCHING

### Branch Basics
```bash
git branch                          # list local branches
git branch -a                       # list all branches (local + remote)
git branch -r                       # list remote branches only
git branch -v                       # branches with last commit
git branch -vv                      # branches with tracking info
git branch --merged                 # branches merged into current
git branch --no-merged              # branches NOT merged into current
git branch --merged main            # branches merged into main

# Create branch
git branch feature-login            # create (don't switch)
git checkout -b feature-login       # create + switch (old way)
git switch -c feature-login         # create + switch (modern)

# Create from specific commit/branch
git branch feature abc1234
git checkout -b feature main
git switch -c feature origin/main

# Rename branch
git branch -m old-name new-name     # rename
git branch -m new-name              # rename current branch
git branch -M old-name new-name     # force rename

# Delete branch
git branch -d feature-login         # delete (only if merged)
git branch -D feature-login         # force delete (even if unmerged)
git push origin --delete feature    # delete remote branch
git push origin :feature            # delete remote branch (shorthand)
```

### Switch Branch
```bash
git checkout main                   # switch to main (old way)
git switch main                     # switch to main (modern, Git 2.23+)
git switch -                        # switch to previous branch
git checkout -                      # switch to previous branch
git switch -c new-branch            # create + switch
git switch --detach abc1234         # detached HEAD at commit
```

### Detached HEAD
```bash
# HEAD points to commit instead of branch
git checkout abc1234                # detached HEAD

# To keep changes, create a branch
git switch -c new-branch
# Or to discard
git switch main
```

---

## 7. MERGING

### Basic Merge
```bash
# Merge feature into main
git switch main
git merge feature-login

# Merge with commit message
git merge feature-login -m "Merge feature-login into main"

# No fast-forward (always create merge commit)
git merge --no-ff feature

# Fast-forward only (fail if not possible)
git merge --ff-only feature

# Squash merge (combine all commits into one staged change)
git merge --squash feature
git commit -m "Add login feature"    # manual commit needed

# Abort merge (during conflict)
git merge --abort
```

### Fast-Forward vs Three-Way Merge
```
Fast-Forward (linear history):
  main:    A --- B --- C
  feature:              \--- D --- E
  After merge: A --- B --- C --- D --- E (main moves forward)

Three-Way Merge (diverged history):
  main:    A --- B --- C --- F
  feature:        \--- D --- E
  After merge: A --- B --- C --- F --- M (M = merge commit)
                       \--- D --- E --/
```

### Merge Conflicts
```bash
# When merge has conflicts:
git merge feature
# CONFLICT (content): Merge conflict in file.txt

# Conflict markers in file:
<<<<<<< HEAD
current branch changes
=======
incoming branch changes
>>>>>>> feature

# Resolve:
# 1. Edit file to resolve conflicts (remove markers)
# 2. Stage resolved files
git add file.txt
# 3. Complete merge
git commit

# Or use merge tool
git mergetool               # open configured merge tool
git mergetool --tool=vscode

# Abort merge
git merge --abort

# See conflicts
git diff --name-only --diff-filter=U    # unmerged files
```

---

## 8. REBASING

### Basic Rebase
```bash
# Rebase feature onto main (replay commits on top of main)
git switch feature
git rebase main

# After rebase, fast-forward main
git switch main
git merge feature    # fast-forward

# Rebase onto specific branch
git rebase main feature    # rebase feature onto main
```

### Rebase vs Merge
```
Before:
  main:    A --- B --- C
  feature:        \--- D --- E

After MERGE:
  main:    A --- B --- C --------- M
  feature:        \--- D --- E ---/

After REBASE:
  main:    A --- B --- C
  feature:               \--- D' --- E'
  (D' and E' are new commits with same changes)
```

### Interactive Rebase (Edit History)
```bash
git rebase -i HEAD~4           # rebase last 4 commits
git rebase -i abc1234          # rebase from specific commit

# Editor opens with:
pick abc1111 First commit
pick abc2222 Second commit
pick abc3333 Third commit
pick abc4444 Fourth commit

# Commands:
# p, pick   = use commit as-is
# r, reword = use commit but edit message
# e, edit   = use commit but stop for amending
# s, squash = merge with previous commit (keep message)
# f, fixup  = merge with previous commit (discard message)
# d, drop   = remove commit
# x, exec   = run shell command

# Common operations:
# Squash last 3 into 1:
pick abc1111 First commit
squash abc2222 Second commit
squash abc3333 Third commit

# Reorder commits: just change the line order

# Drop a commit: change pick to drop (or delete the line)

# Edit commit message: change pick to reword
```

### Rebase Conflicts
```bash
# During rebase conflict:
# 1. Resolve conflicts in files
# 2. Stage resolved files
git add file.txt
# 3. Continue rebase
git rebase --continue

# Skip conflicting commit
git rebase --skip

# Abort rebase
git rebase --abort
```

### Golden Rule
**Never rebase commits that have been pushed to a shared/public branch.**
- Rebase rewrites history — creates new commit hashes.
- Others who pulled the old commits will have conflicts.
- Safe to rebase: local/feature branches before pushing.

---

## 9. STASHING

### Save & Restore Work in Progress
```bash
git stash                            # stash tracked modified + staged files
git stash push -m "WIP: login page"  # stash with description
git stash -u                         # include untracked files
git stash -a                         # include untracked + ignored files
git stash push -m "msg" file.txt     # stash specific file(s)
git stash -p                         # interactive (select hunks to stash)
git stash --keep-index               # stash but keep staged files staged
```

### List & View
```bash
git stash list                       # list all stashes
# stash@{0}: WIP on main: abc1234 commit message
# stash@{1}: On feature: def5678 another message

git stash show                       # show stash diff summary
git stash show -p                    # show stash full diff
git stash show stash@{2}             # show specific stash
git stash show -p stash@{2}          # full diff of specific stash
```

### Apply & Remove
```bash
git stash pop                        # apply latest + remove from stash
git stash apply                      # apply latest (keep in stash)
git stash pop stash@{2}              # apply specific stash + remove
git stash apply stash@{2}            # apply specific (keep)

git stash drop                       # delete latest stash
git stash drop stash@{2}             # delete specific stash
git stash clear                      # delete ALL stashes

# Apply to new branch
git stash branch new-branch          # create branch from stash + apply + drop
git stash branch new-branch stash@{2}
```

---

## 10. UNDOING CHANGES

### Discard Working Directory Changes
```bash
git checkout -- file.txt             # discard changes (old way)
git restore file.txt                 # discard changes (modern, Git 2.23+)
git restore .                        # discard all changes
git restore --source=HEAD~2 file.txt # restore from 2 commits ago
git checkout .                       # discard all changes (old way)
```

### Unstage Files
```bash
git reset HEAD file.txt              # unstage (old way)
git restore --staged file.txt        # unstage (modern)
git restore --staged .               # unstage all
git reset HEAD                       # unstage all (old way)
```

### Amend Last Commit
```bash
git commit --amend                   # edit message + add staged changes
git commit --amend -m "New message"  # just change message
git commit --amend --no-edit         # add staged files, keep message
git commit --amend --author="Name <email>"  # change author
```

### Reset (Move HEAD — affects history)
```bash
# Soft: move HEAD, keep staging + working directory
git reset --soft HEAD~1              # undo last commit (changes remain staged)

# Mixed (default): move HEAD, reset staging, keep working directory
git reset HEAD~1                     # undo last commit (changes become unstaged)
git reset --mixed HEAD~1             # same as above

# Hard: move HEAD, reset staging + working directory (DESTRUCTIVE)
git reset --hard HEAD~1              # undo last commit + discard all changes
git reset --hard origin/main         # reset to remote state
git reset --hard abc1234             # reset to specific commit

# Reset specific file
git reset HEAD file.txt              # unstage file
git reset abc1234 -- file.txt        # reset file to commit version
```

### Revert (Create new commit that undoes — safe for shared history)
```bash
git revert HEAD                      # revert last commit
git revert abc1234                   # revert specific commit
git revert HEAD~3..HEAD              # revert last 3 commits
git revert --no-commit abc1234       # revert without auto-commit
git revert -m 1 abc1234             # revert a merge commit (keep parent 1)
```

### Reset vs Revert
| Aspect | Reset | Revert |
|--------|-------|--------|
| Modifies history | Yes | No (adds new commit) |
| Safe for shared branches | No | Yes |
| Use when | Local/unpushed commits | Pushed/shared commits |

### Clean Untracked Files
```bash
git clean -n                         # dry run — show what would be removed
git clean -f                         # delete untracked files
git clean -fd                        # delete untracked files + directories
git clean -fx                        # delete untracked + ignored files
git clean -fX                        # delete only ignored files
git clean -i                         # interactive mode
```

### Reflog (Safety Net — recover "lost" commits)
```bash
git reflog                           # show all HEAD movements
git reflog show feature              # reflog for specific branch
git reflog --date=relative           # with relative dates

# Recover deleted branch
git reflog
# find the commit hash
git branch recovered-branch abc1234

# Undo hard reset
git reflog
# find commit before reset
git reset --hard abc1234

# Reflog entries expire (default: 90 days)
```

---

## 11. REMOTE REPOSITORIES

### Remote Basics
```bash
git remote                              # list remotes
git remote -v                           # list with URLs
git remote show origin                  # detailed info about remote
git remote add origin https://github.com/user/repo.git
git remote add upstream https://github.com/original/repo.git
git remote rename origin new-name       # rename remote
git remote remove origin                # delete remote
git remote set-url origin https://new-url.git  # change URL
git remote set-url --add origin git@github.com:user/repo.git  # add push URL
git remote prune origin                 # remove stale remote-tracking branches
```

### Fetch (Download without merging)
```bash
git fetch                               # fetch from default remote (origin)
git fetch origin                        # fetch all branches from origin
git fetch origin main                   # fetch specific branch
git fetch --all                         # fetch from all remotes
git fetch --prune                       # fetch + remove stale remote branches
git fetch -p                            # same as --prune
git fetch --tags                        # fetch all tags
```

### Pull (Fetch + Merge/Rebase)
```bash
git pull                                # fetch + merge current branch
git pull origin main                    # pull specific branch
git pull --rebase                       # fetch + rebase (cleaner history)
git pull --rebase=interactive           # fetch + interactive rebase
git pull --ff-only                      # fail if not fast-forward
git pull --no-commit                    # merge without auto-commit
git pull --all                          # pull all branches
```

### Push
```bash
git push                                # push current branch
git push origin main                    # push specific branch
git push -u origin main                 # push + set upstream tracking
git push --set-upstream origin feature  # set upstream for new branch
git push origin --all                   # push all branches
git push origin --tags                  # push all tags
git push origin v1.0.0                  # push specific tag
git push --force                        # force push (DANGEROUS — overwrites remote)
git push --force-with-lease             # safer force push (fails if remote changed)
git push --delete origin feature        # delete remote branch
git push origin :feature                # delete remote branch (shorthand)
```

### Tracking Branches
```bash
# Set upstream
git branch --set-upstream-to=origin/main main
git branch -u origin/main

# See tracking
git branch -vv

# Create tracking branch
git checkout --track origin/feature
git switch --track origin/feature
git checkout -b feature origin/feature
```

### Fork Workflow
```bash
# 1. Fork on GitHub (web UI)
# 2. Clone your fork
git clone https://github.com/you/repo.git

# 3. Add upstream (original repo)
git remote add upstream https://github.com/original/repo.git

# 4. Keep fork updated
git fetch upstream
git switch main
git merge upstream/main
git push origin main

# 5. Create feature branch
git switch -c my-feature

# 6. Push to your fork
git push origin my-feature

# 7. Create Pull Request on GitHub (web UI)
```

---

## 12. TAGGING

### Types of Tags
| Type | Description |
|------|-------------|
| **Lightweight** | Just a pointer to a commit (like a branch that doesn't move) |
| **Annotated** | Full object with tagger info, date, message (recommended) |

### Create Tags
```bash
# Annotated tag (recommended)
git tag -a v1.0.0 -m "Release version 1.0.0"
git tag -a v1.0.0 -m "Release 1.0.0" abc1234    # tag specific commit

# Lightweight tag
git tag v1.0.0
git tag v1.0.0 abc1234                            # tag specific commit

# Signed tag (GPG)
git tag -s v1.0.0 -m "Signed release 1.0.0"
```

### List & View Tags
```bash
git tag                          # list all tags
git tag -l "v1.*"                # list matching pattern
git tag -l --sort=-version:refname  # sort by version (descending)
git show v1.0.0                  # show tag details
git log --oneline --decorate     # see tags in log
```

### Push Tags
```bash
git push origin v1.0.0           # push specific tag
git push origin --tags           # push all tags
git push --follow-tags           # push commits + annotated tags
```

### Delete Tags
```bash
git tag -d v1.0.0                # delete local tag
git push origin --delete v1.0.0  # delete remote tag
git push origin :refs/tags/v1.0.0  # delete remote (shorthand)
```

### Checkout Tag
```bash
git checkout v1.0.0              # detached HEAD at tag
git switch --detach v1.0.0       # same (modern)
git checkout -b hotfix v1.0.0    # create branch from tag
```

---

## 13. CHERRY-PICK

### Pick Specific Commits
```bash
git cherry-pick abc1234                    # apply single commit to current branch
git cherry-pick abc1234 def5678            # multiple commits
git cherry-pick abc1234..def5678           # range (exclusive start)
git cherry-pick abc1234^..def5678          # range (inclusive start)
git cherry-pick --no-commit abc1234        # apply changes without committing
git cherry-pick -x abc1234                 # add "cherry picked from" to message
git cherry-pick -e abc1234                 # edit commit message
git cherry-pick --abort                    # abort during conflict
git cherry-pick --continue                 # continue after resolving conflict
git cherry-pick --skip                     # skip current commit and continue
```

---

## 14. ADVANCED COMMANDS

### Bisect (Binary Search for Bug)
```bash
git bisect start
git bisect bad                     # current commit is bad
git bisect good abc1234            # known good commit
# Git checks out middle commit — test it, then:
git bisect bad                     # if still broken
git bisect good                    # if working
# Repeat until Git finds the first bad commit
git bisect reset                   # return to original state

# Automated bisect
git bisect start HEAD abc1234
git bisect run ./test-script.sh    # auto-run test at each step
```

### Worktree (Multiple Working Directories)
```bash
git worktree add ../hotfix-branch hotfix    # check out branch in separate folder
git worktree add ../new-feature -b feature  # create + checkout in new folder
git worktree list                            # list all worktrees
git worktree remove ../hotfix-branch        # remove worktree
git worktree prune                           # clean up stale worktrees
```

### Submodules
```bash
# Add submodule
git submodule add https://github.com/user/lib.git libs/lib

# Clone with submodules
git clone --recurse-submodules https://github.com/user/repo.git

# Initialize submodules (if cloned without --recurse)
git submodule init
git submodule update
# or combined:
git submodule update --init --recursive

# Update submodules to latest
git submodule update --remote

# Remove submodule
git submodule deinit libs/lib
git rm libs/lib
rm -rf .git/modules/libs/lib
```

### grep (Search in Repository)
```bash
git grep "TODO"                        # search working directory
git grep -n "TODO"                     # with line numbers
git grep -c "TODO"                     # count per file
git grep -l "TODO"                     # filenames only
git grep "TODO" HEAD~5                 # search at specific commit
git grep -i "todo"                     # case-insensitive
git grep -e "pattern1" --and -e "pattern2"  # AND
git grep -e "pattern1" --or -e "pattern2"   # OR
git grep "TODO" -- '*.java'            # search specific file types
```

### Archive (Create Release Package)
```bash
git archive --format=zip HEAD -o release.zip
git archive --format=tar.gz --prefix=my-project/ HEAD -o release.tar.gz
git archive --format=zip v1.0.0 -o v1.0.0.zip      # from tag
```

### Patch
```bash
# Create patches
git format-patch HEAD~3              # create patches for last 3 commits
git format-patch main..feature       # patches for branch difference
git diff > changes.patch             # simple diff patch

# Apply patches
git apply changes.patch              # apply diff patch
git am 0001-commit-msg.patch         # apply format-patch (with commit)
git am *.patch                       # apply multiple patches
```

---

## 15. GIT HOOKS

### What are Hooks?
- Scripts that run **automatically** at Git events.
- Located in `.git/hooks/` (local, not shared) or via core.hooksPath.
- Must be executable (`chmod +x`).

### Available Hooks
| Hook | Runs Before/After | Use Case |
|------|-------------------|----------|
| `pre-commit` | Before commit | Lint, format, test |
| `prepare-commit-msg` | After default message | Modify commit message template |
| `commit-msg` | After message entered | Validate commit message format |
| `post-commit` | After commit | Notifications |
| `pre-push` | Before push | Run tests, check branch |
| `pre-rebase` | Before rebase | Prevent rebasing published commits |
| `post-merge` | After merge | Install dependencies |
| `post-checkout` | After checkout | Setup environment |
| `pre-receive` | Before push accepted (server) | Enforce policies |
| `post-receive` | After push accepted (server) | Deploy, notify |

### Example Hooks
```bash
# .git/hooks/pre-commit
#!/bin/bash
# Run tests before committing
mvn test
if [ $? -ne 0 ]; then
    echo "Tests failed. Commit aborted."
    exit 1
fi

# .git/hooks/commit-msg
#!/bin/bash
# Enforce conventional commit format
if ! grep -qE "^(feat|fix|docs|style|refactor|test|chore|perf|ci|build|revert)\(?.+\)?: .+" "$1"; then
    echo "ERROR: Commit message must follow conventional format"
    echo "Example: feat(auth): add login feature"
    exit 1
fi
```

### Shared Hooks
```bash
# Store hooks in repo
mkdir .githooks
# create hooks in .githooks/

# Configure Git to use them
git config core.hooksPath .githooks

# Or project-level
echo "core.hooksPath=.githooks" >> .git/config
```

### Popular Hook Tools
- **Husky** — Git hooks for Node.js projects
- **pre-commit** — Python-based hook framework
- **lefthook** — Fast polyglot Git hook manager
- **lint-staged** — Run linters on staged files

---

## 16. GIT WORKFLOWS

### Git Flow
```
main (production) ←── release branches ←── develop ←── feature branches
                  ←── hotfix branches

Branches:
  main      — production-ready code
  develop   — integration branch
  feature/* — new features (from develop)
  release/* — prepare release (from develop)
  hotfix/*  — urgent fixes (from main)

Flow:
  1. feature/login → develop → release/1.0 → main (+ tag v1.0)
  2. hotfix/bug → main (+ tag v1.0.1) + merge back to develop
```

### GitHub Flow (Simpler)
```
main ←── feature branches

Flow:
  1. Create branch from main
  2. Make changes + commits
  3. Open Pull Request
  4. Code review + discussion
  5. Merge to main
  6. Deploy from main
```

### Trunk-Based Development
```
main ←── short-lived feature branches (< 1-2 days)

Flow:
  1. Pull latest main
  2. Create short-lived branch
  3. Small, frequent commits
  4. Merge to main quickly
  5. Feature flags for incomplete features
  6. CI/CD deploys from main
```

---

## 17. SSH SETUP

### Generate SSH Key
```bash
ssh-keygen -t ed25519 -C "ritesh@example.com"
# or for older systems:
ssh-keygen -t rsa -b 4096 -C "ritesh@example.com"

# Follow prompts (default location: ~/.ssh/id_ed25519)
```

### Add to SSH Agent
```bash
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_ed25519
```

### Add Public Key to GitHub/GitLab
```bash
# Copy public key
cat ~/.ssh/id_ed25519.pub
# Paste in: GitHub → Settings → SSH Keys → New SSH Key
```

### Test Connection
```bash
ssh -T git@github.com
# "Hi username! You've successfully authenticated..."
```

### Switch Remote from HTTPS to SSH
```bash
git remote set-url origin git@github.com:user/repo.git
```

### SSH Config (~/.ssh/config)
```
Host github.com
    HostName github.com
    User git
    IdentityFile ~/.ssh/id_ed25519
    AddKeysToAgent yes

Host gitlab.com
    HostName gitlab.com
    User git
    IdentityFile ~/.ssh/id_ed25519_gitlab

Host work-github
    HostName github.com
    User git
    IdentityFile ~/.ssh/id_ed25519_work
```

---

## 18. .GITATTRIBUTES

```bash
# .gitattributes — control how Git handles files

# Auto-detect text files and normalize line endings
* text=auto

# Force specific line endings
*.sh text eol=lf
*.bat text eol=crlf
*.cmd text eol=crlf

# Binary files (don't diff/merge)
*.png binary
*.jpg binary
*.gif binary
*.ico binary
*.zip binary
*.jar binary
*.pdf binary
*.woff binary
*.woff2 binary

# Mark as generated (collapse in PR diffs)
*.min.js linguist-generated=true
*.min.css linguist-generated=true
package-lock.json linguist-generated=true

# Linguist language detection
*.java linguist-language=Java
docs/** linguist-documentation=true
vendor/** linguist-vendored=true

# Custom diff driver
*.md diff=markdown
*.docx diff=word

# Merge strategy
database.xml merge=ours
```

---

## 19. BEST PRACTICES

### Commits
```
1. Make small, focused commits (one logical change per commit)
2. Write meaningful commit messages (conventional commits)
3. Don't commit generated files (add to .gitignore)
4. Don't commit secrets/credentials
5. Test before committing
6. Use present tense: "Add feature" not "Added feature"
```

### Branching
```
1. Keep branches short-lived
2. Delete merged branches
3. Use descriptive names: feature/user-auth, fix/login-bug
4. Protect main branch (require PR reviews)
5. Keep main always deployable
6. Pull/rebase frequently to avoid large merge conflicts
```

### Collaboration
```
1. Pull before push
2. Use Pull Requests for code review
3. Never force push to shared branches
4. Use --force-with-lease instead of --force when necessary
5. Resolve conflicts locally before pushing
6. Keep commit history clean (squash WIP commits)
```

### Security
```
1. Never commit passwords, API keys, tokens
2. Use .gitignore for .env files
3. Use git-secrets or pre-commit hooks to prevent secret commits
4. Use SSH keys instead of HTTPS passwords
5. Sign commits with GPG (git commit -S)
6. Audit with: git log --all --full-history -- '**/secret*'
```

---

## 20. QUICK REFERENCE CHEAT SHEET

### Daily Workflow
```bash
git status                    # check status
git add .                     # stage all changes
git commit -m "message"       # commit
git push                      # push to remote
git pull --rebase             # get latest from remote
```

### Branch Workflow
```bash
git switch -c feature         # create + switch
# ... make changes ...
git add . && git commit -m "feat: add feature"
git push -u origin feature    # push new branch
# Create PR on GitHub
git switch main               # switch back
git pull                      # get merged changes
git branch -d feature         # delete local branch
```

### Oops Commands
```bash
git commit --amend            # fix last commit
git restore file.txt          # discard changes
git restore --staged file.txt # unstage file
git reset --soft HEAD~1       # undo commit (keep changes staged)
git reset HEAD~1              # undo commit (keep changes unstaged)
git reset --hard HEAD~1       # undo commit + discard changes
git revert HEAD               # undo commit safely (new commit)
git stash                     # save work temporarily
git stash pop                 # restore saved work
git reflog                    # find "lost" commits
```

### Info Commands
```bash
git log --oneline --graph --all   # visual history
git diff                          # see changes
git diff --staged                 # see staged changes
git blame file.txt                # who changed what
git show abc1234                  # show commit details
git remote -v                     # see remotes
git branch -vv                    # see branches + tracking
```

### Key Concepts
```
Working Dir → git add → Staging → git commit → Local Repo → git push → Remote Repo
Remote Repo → git fetch → Local Repo → git merge → Working Dir
Remote Repo → git pull → Working Dir (fetch + merge combined)

HEAD     = pointer to current branch/commit
origin   = default name for remote repository
main     = default primary branch
upstream = original repo (in fork workflow)
```

### Comparison Table
| Action | Old Command | Modern Command |
|--------|-------------|----------------|
| Switch branch | `git checkout main` | `git switch main` |
| Create + switch | `git checkout -b feat` | `git switch -c feat` |
| Discard changes | `git checkout -- file` | `git restore file` |
| Unstage | `git reset HEAD file` | `git restore --staged file` |

---

*Complete Git Reference — All 20 Topics*
*Last updated: April 2026*
