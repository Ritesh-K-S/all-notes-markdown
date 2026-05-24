# Chapter 32: Azure DevOps - Repos

---

## Table of Contents

- [Overview](#overview)
- [Part 1: Azure Repos Fundamentals](#part-1-azure-repos-fundamentals)
- [Part 2: Setting Up Azure DevOps (Portal Walkthrough)](#part-2-setting-up-azure-devops-portal-walkthrough)
- [Part 3: Git Repositories](#part-3-git-repositories)
- [Part 4: Branch Policies](#part-4-branch-policies)
- [Part 5: Pull Requests & Code Review](#part-5-pull-requests--code-review)
- [Part 6: Security & Permissions](#part-6-security--permissions)
- [Part 7: az CLI Reference](#part-7-az-cli-reference)
- [Quick Reference](#quick-reference)
- [What's Next?](#whats-next)

---

## Overview

Azure DevOps Repos provides Git repositories for source control. It's like GitHub but integrated into the Azure DevOps ecosystem with built-in CI/CD pipelines, work item tracking, and artifact management. You get unlimited private Git repos, branch policies, pull request workflows, and code review tools.

```
What you'll learn:
├── Azure Repos Fundamentals
│   ├── What is Azure DevOps (Repos, Pipelines, Boards, Artifacts)
│   ├── Azure Repos vs GitHub
│   └── Organization → Project → Repository hierarchy
├── Setting Up Azure DevOps Organization & Project
├── Git Repositories (create, clone, push)
├── Branch Policies (protect main branch)
├── Pull Requests & Code Review
├── Security & Permissions
└── az CLI reference
```

---

## Part 1: Azure Repos Fundamentals

```
┌─────────────────────────────────────────────────────────────────────┐
│           AZURE DEVOPS OVERVIEW                                        │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Azure DevOps is a suite of developer services:                      │
│                                                                       │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │ Azure DevOps Organization: mycompany.visualstudio.com       │  │
│ │                                                              │  │
│ │ ├── Project: myapp                                         │  │
│ │ │   ├── Repos (Git source control) ← This chapter        │  │
│ │ │   ├── Pipelines (CI/CD) ← Next chapter                 │  │
│ │ │   ├── Boards (work items, sprints, kanban)             │  │
│ │ │   ├── Test Plans (manual/automated testing)            │  │
│ │ │   └── Artifacts (package management) ← Chapter 34      │  │
│ │ │                                                           │  │
│ │ ├── Project: infrastructure                                │  │
│ │ │   ├── Repos                                              │  │
│ │ │   └── Pipelines                                          │  │
│ │ │                                                           │  │
│ │ └── Project: mobile-app                                    │  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
│ Azure Repos vs GitHub:                                               │
│ ┌──────────────────┬──────────────────┬──────────────────────┐    │
│ │ Feature          │ Azure Repos      │ GitHub               │    │
│ ├──────────────────┼──────────────────┼──────────────────────┤    │
│ │ Git hosting      │ Yes ✅          │ Yes ✅              │    │
│ │ Private repos    │ Unlimited (free) │ Unlimited (free)    │    │
│ │ Branch policies  │ Built-in ✅     │ Built-in ✅         │    │
│ │ CI/CD            │ Azure Pipelines  │ GitHub Actions       │    │
│ │ Work tracking    │ Azure Boards     │ GitHub Issues        │    │
│ │ Packages         │ Azure Artifacts  │ GitHub Packages      │    │
│ │ Community        │ Enterprise-focused│ Largest community  │    │
│ │ Best for         │ Enterprise Azure │ Open source + general│    │
│ └──────────────────┴──────────────────┴──────────────────────┘    │
│                                                                       │
│ ⚡ Both work great! Use Azure Repos if your company is            │
│   all-in on Azure DevOps. Use GitHub for everything else.        │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 2: Setting Up Azure DevOps (Portal Walkthrough)

```
Go to: https://dev.azure.com

┌─────────────────────────────────────────────────────────────────────┐
│           CREATE ORGANIZATION                                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ 1. Sign in with Microsoft account                                   │
│ 2. Create organization:                                              │
│    Organization name: [mycompany]                                  │
│    URL: https://dev.azure.com/mycompany                            │
│    Region: [Central India ▼]                                      │
│                                                                       │
│ 3. Create project:                                                   │
│    Project name: [myapp]                                           │
│    Description: [My application project]                           │
│    Visibility: ● Private  ○ Public                               │
│    Version control: ● Git  ○ TFVC (legacy)                      │
│    Work item process: [Agile ▼] (Agile/Scrum/CMMI/Basic)       │
│                                                                       │
│ [Create project]                                                     │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 3: Git Repositories

```
┌─────────────────────────────────────────────────────────────────────┐
│           GIT REPOSITORIES                                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Project → Repos → Files                                           │
│                                                                       │
│ Create/initialize repo:                                              │
│ ├── Empty repo → shows clone URL and instructions                │
│ ├── Add .gitignore: [Node ▼]                                    │
│ ├── Add README: ☑                                                │
│ └── [Initialize]                                                  │
│                                                                       │
│ Clone URL:                                                           │
│ HTTPS: https://mycompany@dev.azure.com/mycompany/myapp/_git/myapp│
│ SSH: git@ssh.dev.azure.com:v3/mycompany/myapp/myapp             │
│                                                                       │
│ Clone and push:                                                      │
│ git clone https://dev.azure.com/mycompany/myapp/_git/myapp       │
│ cd myapp                                                             │
│ # make changes                                                       │
│ git add . && git commit -m "feat: initial commit"                │
│ git push origin main                                                │
│                                                                       │
│ Multiple repos per project:                                          │
│ Repos → [+ New repository]                                       │
│ Name: [myapp-frontend]                                             │
│ ⚡ One project can have many repos (monorepo or multi-repo).     │
│                                                                       │
│ Import from external:                                                │
│ Repos → Import → Clone URL: https://github.com/user/repo.git   │
│ ⚡ Migrate from GitHub, Bitbucket, or any Git URL.               │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 4: Branch Policies

```
┌─────────────────────────────────────────────────────────────────────┐
│           BRANCH POLICIES                                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Project → Repos → Branches → main → ⋮ → Branch policies       │
│                                                                       │
│ ☑ Require minimum number of reviewers: [2]                       │
│   ☑ Allow requestors to approve their own changes: ☐            │
│   ☑ Prohibit most recent pusher from approving: ☑              │
│   ☑ Reset code reviewer votes when there are new changes: ☑   │
│                                                                       │
│ ☑ Check for linked work items: Required                          │
│   ⚡ Every PR must link to a work item (user story/bug).        │
│                                                                       │
│ ☑ Check for comment resolution: Required                         │
│   ⚡ All PR comments must be resolved before merge.             │
│                                                                       │
│ ☑ Limit merge types:                                              │
│   ☑ Squash merge (recommended — clean history)                 │
│   ☐ Basic merge (merge commit)                                  │
│   ☐ Rebase                                                       │
│   ☐ Rebase with merge                                           │
│                                                                       │
│ ☑ Build validation:                                                │
│   Pipeline: [Build Pipeline ▼]                                   │
│   Trigger: Automatic                                               │
│   Policy requirement: Required                                    │
│   ⚡ PR build must pass before merge is allowed!                │
│                                                                       │
│ ☑ Status checks:                                                   │
│   Add external status checks (SonarQube, security scans, etc.) │
│                                                                       │
│ [Save changes]                                                       │
│                                                                       │
│ ⚡ With policies, nobody can push directly to main!              │
│   All changes MUST go through a Pull Request.                    │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 5: Pull Requests & Code Review

```
┌─────────────────────────────────────────────────────────────────────┐
│           PULL REQUESTS                                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Create PR:                                                           │
│ Repos → Pull requests → New pull request                        │
│                                                                       │
│ Source branch: [feature/user-auth]                                  │
│ Target branch: [main]                                               │
│ Title: [Add user authentication with JWT]                          │
│ Description: [Implements JWT-based auth with refresh tokens...]   │
│ Reviewers: [team-leads ▼]                                        │
│ Work items: [#1234 - Implement user authentication]               │
│ Labels: [enhancement]                                               │
│                                                                       │
│ [Create]                                                             │
│                                                                       │
│ PR Review workflow:                                                  │
│ 1. Developer creates PR from feature branch → main               │
│ 2. Build validation runs automatically (must pass) ✅             │
│ 3. Reviewers are notified (email + Teams notification)           │
│ 4. Reviewers comment on code, request changes                    │
│ 5. Developer addresses feedback, pushes fixes                     │
│ 6. Reviewers approve (required minimum met)                       │
│ 7. All comments resolved, all checks pass                        │
│ 8. [Complete] → Squash merge into main                          │
│ 9. Source branch auto-deleted (optional)                          │
│                                                                       │
│ Review options:                                                      │
│ ├── Approve                                                       │
│ ├── Approve with suggestions                                     │
│ ├── Wait for author                                               │
│ ├── Reject                                                        │
│ └── Inline comments on specific lines of code                   │
│                                                                       │
│ Complete options:                                                    │
│ ├── Merge type: Squash / Merge / Rebase                         │
│ ├── ☑ Delete source branch after merging                        │
│ ├── ☑ Complete associated work items                            │
│ └── Auto-complete: Auto-merge when all policies are met         │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 6: Security & Permissions

```
Security levels:
├── Organization: Organization Settings → Permissions
│   Groups: Project Collection Administrators, Contributors
│
├── Project: Project Settings → Permissions
│   Groups: Project Administrators, Contributors, Readers
│
├── Repository: Repos → Settings → Security (per repo)
│   ├── Contribute (push code)
│   ├── Create branch
│   ├── Create tag
│   ├── Manage permissions
│   ├── Force push (dangerous — usually deny!)
│   └── Bypass policies (for emergencies only)
│
└── Branch-level: Per-branch permissions
    ├── Lock branch (nobody can push)
    └── Per-user/group overrides
```

---

## Part 7: az CLI Reference

```bash
# Install Azure DevOps extension
az extension add --name azure-devops

# Login and set defaults
az devops configure --defaults organization=https://dev.azure.com/mycompany project=myapp

# Create repository
az repos create --name myapp-backend

# List repositories
az repos list --output table

# Create branch
az repos ref create --name refs/heads/feature/new-feature --repository myapp --object-id <commit-sha>

# Create pull request
az repos pr create \
  --repository myapp \
  --source-branch feature/new-feature \
  --target-branch main \
  --title "Add new feature" \
  --description "Implements feature X"

# List pull requests
az repos pr list --repository myapp --output table

# Set branch policy (require 2 reviewers)
az repos policy approver-count create \
  --repository-id <repo-id> \
  --branch main \
  --minimum-approver-count 2 \
  --enabled true

# Delete repository
az repos delete --id <repo-id> --yes
```

---

## Real-World Patterns

### Pattern 1: Trunk-Based Development with PR Policies

```
┌─────────────────────────────────────────────────┐
│       Trunk-Based Development Flow              │
├─────────────────────────────────────────────────┤
│                                                 │
│  feature/login ──→ Pull Request ──→ main        │
│                      │                          │
│              ┌───────┴───────┐                    │
│              │ PR Policies   │                    │
│              ├───────────────┤                    │
│              │ ✓ 2 reviewers  │                    │
│              │ ✓ Build passes │                    │
│              │ ✓ Tests pass   │                    │
│              │ ✓ No conflicts │                    │
│              └───────────────┘                    │
│                                                 │
│  Short-lived branches merged within 1-2 days   │
└─────────────────────────────────────────────────┘
```

---

## Quick Reference

```
Azure DevOps URL: https://dev.azure.com/{organization}
Hierarchy: Organization → Project → Repository

Repos = Git repositories (unlimited private, free)
Branch Policies = Protect main (require PR, reviewers, build)
Pull Requests = Code review workflow before merge

Key policies:
  Minimum reviewers (2 recommended)
  Build validation (PR must pass CI)
  Comment resolution required
  Linked work items required
  Squash merge (clean history)
```

---

## What's Next?

Next chapter: [Chapter 33: Azure DevOps - Pipelines](33-azure-devops-pipelines.md) — CI/CD with YAML pipelines, stages, approvals, and deployments.
