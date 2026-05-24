# Chapter 30: CodeCommit

---

## Table of Contents

- [Overview](#overview)
- [Part 1: CodeCommit Fundamentals](#part-1-codecommit-fundamentals)
- [Part 2: Creating a Repository (Full Portal Walkthrough)](#part-2-creating-a-repository-full-portal-walkthrough)
- [Part 3: Branches, Pull Requests & Code Review](#part-3-branches-pull-requests--code-review)
- [Part 4: Security & Access Control](#part-4-security--access-control)
- [Part 5: Notifications & Triggers](#part-5-notifications--triggers)
- [Part 6: Terraform & CLI Examples](#part-6-terraform--cli-examples)
- [Part 7: Real-World Patterns](#part-7-real-world-patterns)
- [Quick Reference](#quick-reference)
- [What's Next?](#whats-next)

---

## Overview

### What is Version Control? Why Do We Need It?

**Version control** is like Google Docs' version history for your code. It tracks every change, who made it, and when. If something breaks, you can go back to a working version. When multiple developers work on the same project, version control prevents them from overwriting each other's work.

**Git** is the most popular version control system. Services like **GitHub**, **GitLab**, and **Bitbucket** host your Git repositories in the cloud. AWS's version was **CodeCommit** — a private Git hosting service integrated with IAM and other AWS services.

⚠️ **Important:** AWS CodeCommit stopped accepting new customers on July 25, 2024. Existing customers can continue using it. For new projects, consider **GitHub**, **GitLab**, or **Bitbucket**. However, CodeCommit still appears on AWS certification exams.

```
What you'll learn:
├── CodeCommit Fundamentals
├── Repository creation & management
├── Branches, PRs, and code review
├── Access control (IAM + SSH/HTTPS)
├── Notifications & triggers
├── CLI & Terraform examples
└── Real-world patterns
```

---

## Part 1: CodeCommit Fundamentals

```
┌─────────────────────────────────────────────────────────────────────┐
│           AWS CODECOMMIT                                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What: Managed, secure, private Git repositories hosted in AWS.    │
│ Like GitHub/GitLab but fully integrated with AWS IAM.             │
│                                                                       │
│ Why it existed:                                                      │
│ ├── Private repos with AWS IAM authentication                    │
│ ├── Encrypted at rest (KMS) and in transit (HTTPS/SSH)          │
│ ├── No size limits on repos                                       │
│ ├── High availability (multi-AZ)                                 │
│ ├── Integrates natively with CodeBuild, CodePipeline, etc.      │
│ └── Data stays in your AWS account (compliance)                  │
│                                                                       │
│ Comparison:                                                          │
│ ┌──────────────┬──────────┬──────────┬──────────────────────────┐ │
│ │ Feature      │CodeCommit│ GitHub   │ GitLab                  │ │
│ ├──────────────┼──────────┼──────────┼──────────────────────────┤ │
│ │ Authentication│ IAM     │ PAT/SSH  │ PAT/SSH                │ │
│ │ Pricing      │ Free/low │ Free+paid│ Free+paid              │ │
│ │ CI/CD builtin│ No       │ Actions  │ CI/CD                  │ │
│ │ Community    │ None     │ Huge     │ Large                  │ │
│ │ AWS native   │ Yes      │ Via conn.│ Via connection          │ │
│ │ Status       │ Sunset   │ Active   │ Active                 │ │
│ └──────────────┴──────────┴──────────┴──────────────────────────┘ │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 2: Creating a Repository (Full Portal Walkthrough)

```
Console → CodeCommit → Repositories → Create repository

┌─────────────────────────────────────────────────────────────────┐
│           CREATE REPOSITORY                                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Repository name: [my-application]                              │
│ → Must be unique within AWS account per region                │
│ → Alphanumeric, hyphens, underscores, dots                   │
│                                                                   │
│ Description: [Main application repository]                     │
│ → Optional but recommended                                    │
│                                                                   │
│ Tags:                                                           │
│ Key: [team]  Value: [backend]                                 │
│                                                                   │
│ ── Additional configuration ──                                 │
│                                                                   │
│ ☑ Enable Amazon CodeGuru Reviewer                             │
│ → Automated code reviews using ML                             │
│ → Finds bugs, security issues, best practices                │
│ → Additional charges apply                                    │
│                                                                   │
│                    [Create]                                      │
└─────────────────────────────────────────────────────────────────┘

After creation — Connect to repository:

┌─────────────────────────────────────────────────────────────────┐
│ Connection method:                                               │
│                                                                   │
│ ── HTTPS (GRC) — ⚡ Recommended ──                             │
│ Uses git-remote-codecommit (Python helper)                     │
│ pip install git-remote-codecommit                              │
│ git clone codecommit://my-application                          │
│ → Uses AWS CLI credentials automatically                      │
│                                                                   │
│ ── HTTPS ──                                                     │
│ Generate HTTPS credentials in IAM:                             │
│ IAM → Users → Security credentials → CodeCommit credentials  │
│ git clone https://git-codecommit.us-east-1.amazonaws.com/     │
│           v1/repos/my-application                              │
│                                                                   │
│ ── SSH ──                                                       │
│ Upload SSH public key to IAM:                                  │
│ IAM → Users → Security credentials → SSH keys for CodeCommit │
│ git clone ssh://APKAEXAMPLE@git-codecommit.us-east-1.         │
│           amazonaws.com/v1/repos/my-application                │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## Part 3: Branches, Pull Requests & Code Review

```
┌─────────────────────────────────────────────────────────────────────┐
│           BRANCHES & PULL REQUESTS                                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Console → CodeCommit → Repositories → [repo] → Branches           │
│                                                                       │
│ Create branch:                                                       │
│ Branch name: [feature/user-auth]                                   │
│ Branch from: [main ▼]                                              │
│ Or from commit ID: [optional]                                      │
│                                                                       │
│ Pull Requests:                                                       │
│ Console → [repo] → Pull requests → Create pull request            │
│                                                                       │
│ Source: [feature/user-auth ▼]                                      │
│ Destination: [main ▼]                                              │
│ Title: [Add user authentication module]                            │
│ Description: [Implements JWT-based auth...]                        │
│                                                                       │
│ PR features:                                                         │
│ ├── Diff view (side-by-side or unified)                          │
│ ├── Line-by-line comments                                         │
│ ├── Approval rules (require N approvals before merge)            │
│ ├── Merge strategies: Fast-forward, Squash, 3-way                │
│ └── Activity log (all comments, approvals)                       │
│                                                                       │
│ Approval rules:                                                      │
│ Console → [repo] → Settings → Approval rule templates             │
│ Rule name: [require-2-approvals]                                   │
│ Number of approvals: [2]                                           │
│ Approval pool: [arn:aws:iam::123456:role/developers]              │
│ Branch filter: [main]                                              │
│ → Enforces code review before merging to main                    │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 4: Security & Access Control

```
┌─────────────────────────────────────────────────────────────────────┐
│           SECURITY & ACCESS CONTROL                                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ IAM Policies:                                                        │
│ ├── AWSCodeCommitFullAccess: Full access to all repos            │
│ ├── AWSCodeCommitPowerUser: All actions except delete repo       │
│ ├── AWSCodeCommitReadOnly: Read access only                      │
│ └── Custom policies for per-repo access                          │
│                                                                       │
│ Restrict branch pushes (IAM condition):                             │
│ {                                                                    │
│   "Effect": "Deny",                                                │
│   "Action": [                                                       │
│     "codecommit:GitPush",                                          │
│     "codecommit:DeleteBranch",                                     │
│     "codecommit:PutFile",                                          │
│     "codecommit:MergeBranchesByFastForward"                       │
│   ],                                                                 │
│   "Resource": "arn:aws:codecommit:*:*:my-repo",                   │
│   "Condition": {                                                    │
│     "StringEqualsIfExists": {                                      │
│       "codecommit:References": ["refs/heads/main"]                │
│     }                                                                │
│   }                                                                  │
│ }                                                                    │
│ → Prevents direct push to main (forces PR workflow)              │
│                                                                       │
│ Encryption:                                                          │
│ ├── At rest: AWS managed key or customer managed KMS key        │
│ └── In transit: HTTPS or SSH                                     │
│                                                                       │
│ Cross-account access:                                               │
│ ├── Use IAM roles + assume-role                                  │
│ └── Configure git credential helper for cross-account            │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 5: Notifications & Triggers

```
┌─────────────────────────────────────────────────────────────────────┐
│           NOTIFICATIONS & TRIGGERS                                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Notifications (Console → [repo] → Notify → Create notification): │
│ Name: [repo-notifications]                                         │
│ Events:                                                              │
│ ☑ On branches and tags: Created, Updated, Deleted                │
│ ☑ On pull requests: Created, Updated, Merged, Closed            │
│ ☑ On comments on pull requests                                   │
│ ☑ On comments on commits                                         │
│ Target: SNS topic or AWS Chatbot (Slack)                          │
│                                                                       │
│ Triggers (Console → [repo] → Settings → Triggers):               │
│ Trigger name: [build-trigger]                                      │
│ Events: ● All repository events                                   │
│         ○ Push to existing branch                                  │
│         ○ Create branch or tag                                     │
│         ○ Delete branch or tag                                     │
│ Branches: [main, develop] (or all)                                │
│ Target: SNS topic or Lambda function                               │
│                                                                       │
│ ⚡ Use triggers to start CodePipeline or Lambda on push          │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 6: Terraform & CLI Examples

```hcl
resource "aws_codecommit_repository" "app" {
  repository_name = "my-application"
  description     = "Main application repository"
  default_branch  = "main"
  tags = { Team = "backend" }
}

resource "aws_codecommit_approval_rule_template" "two_approvals" {
  name        = "require-2-approvals"
  description = "Require 2 approvals before merge"
  content     = jsonencode({
    Version               = "2018-11-08"
    DestinationReferences = ["refs/heads/main"]
    Statements = [{
      Type                    = "Approvers"
      NumberOfApprovalsNeeded = 2
      ApprovalPoolMembers     = ["arn:aws:iam::123456789:role/developers"]
    }]
  })
}
```

```bash
# Create repository
aws codecommit create-repository \
  --repository-name my-application \
  --repository-description "Main application"

# List repositories
aws codecommit list-repositories

# Clone
git clone codecommit://my-application

# Create branch
aws codecommit create-branch \
  --repository-name my-application \
  --branch-name feature/new-api \
  --commit-id abc123def456

# Create pull request
aws codecommit create-pull-request \
  --title "Add new API endpoints" \
  --targets repositoryName=my-application,sourceReference=feature/new-api,destinationReference=main
```

---

## Part 7: Real-World Patterns

```
┌─────────────────────────────────────────────────────────────────────┐
│           REAL-WORLD PATTERNS                                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Pattern: CodeCommit + CodePipeline CI/CD                            │
│ ┌──────────┐   ┌───────────┐   ┌───────────┐   ┌─────────────┐  │
│ │CodeCommit│──►│CodeBuild  │──►│CodeDeploy │──►│ Production  │  │
│ │(git push)│   │(build/test)│  │(deploy)   │   │(EC2/ECS/etc)│  │
│ └──────────┘   └───────────┘   └───────────┘   └─────────────┘  │
│                                                                       │
│ Pattern: Cross-Region Replication                                   │
│ ├── Replicate repos to another region for DR                     │
│ ├── Use Lambda trigger + CodeCommit API                          │
│ └── Or use git push to mirror repository                         │
│                                                                       │
│ Pattern: Migrate from GitHub to CodeCommit                          │
│ git clone --mirror https://github.com/user/repo.git             │
│ cd repo.git                                                        │
│ git push codecommit://my-repo --all                               │
│ git push codecommit://my-repo --tags                              │
│                                                                       │
│ ⚡ For new projects: Use GitHub + CodeStar Connections            │
│   (since CodeCommit is sunset)                                     │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Quick Reference

```
CodeCommit Quick Reference:
├── ⚠️ Sunset: No new customers after July 2024
├── What: Managed private Git repositories
├── Auth: IAM (HTTPS via GRC or credentials, SSH)
├── Encryption: At rest (KMS) + in transit
├── PRs: Approval rules, diff view, comments
├── Triggers: Push events → Lambda or SNS
├── Notifications: SNS or AWS Chatbot (Slack)
├── Alternative: GitHub + CodeStar Connections
└── ⚡ Still on AWS certification exams
```

---

## What's Next?

In **Chapter 31: CodeBuild**, we'll cover AWS's managed build service — build projects, buildspec.yml, build environments, caching, and artifact management.
