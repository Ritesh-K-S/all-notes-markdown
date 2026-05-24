# Chapter 30: Cloud Source Repositories

---

## Table of Contents

- [Overview](#overview)
- [Part 1: Cloud Source Repositories Fundamentals](#part-1-cloud-source-repositories-fundamentals)
- [Part 2: Architecture & How It Works](#part-2-architecture--how-it-works)
- [Part 3: Creating a Repository](#part-3-creating-a-repository)
- [Part 4: Mirroring from GitHub](#part-4-mirroring-from-github)
- [Part 5: Mirroring from Bitbucket](#part-5-mirroring-from-bitbucket)
- [Part 6: Authentication & Access](#part-6-authentication--access)
- [Part 7: Cloning & Working with Repos](#part-7-cloning--working-with-repos)
- [Part 8: Browsing Code in Console](#part-8-browsing-code-in-console)
- [Part 9: Search Across Repositories](#part-9-search-across-repositories)
- [Part 10: Integration with Cloud Build](#part-10-integration-with-cloud-build)
- [Part 11: Integration with Other GCP Services](#part-11-integration-with-other-gcp-services)
- [Part 12: IAM & Security](#part-12-iam--security)
- [Part 13: Notifications & Pub/Sub](#part-13-notifications--pubsub)
- [Part 14: Terraform & CLI](#part-14-terraform--cli)
- [Part 15: Real-World Patterns](#part-15-real-world-patterns)
- [Quick Reference](#quick-reference)
- [What's Next?](#whats-next)

---

## Overview

Cloud Source Repositories (CSR) is Google Cloud's fully managed, private Git repository service. It lets you host Git repos directly in GCP or mirror existing repos from GitHub and Bitbucket — giving you a single pane of glass for browsing, searching, and integrating your source code with GCP services like Cloud Build, Cloud Functions, and App Engine. Think of it as "your Git repos, inside GCP" with native integrations to the rest of the platform.

```
┌─────────────────────────────────────────────────────────────────────┐
│ WHAT YOU'LL LEARN                                                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ ├── 1. What is CSR and when to use it vs GitHub/GitLab             │
│ ├── 2. Architecture — how repos, mirrors, and projects connect     │
│ ├── 3. Creating a hosted repository                                │
│ ├── 4. Mirroring from GitHub (automatic sync)                      │
│ ├── 5. Mirroring from Bitbucket (automatic sync)                   │
│ ├── 6. Authentication methods (SSH, gcloud, manual credentials)    │
│ ├── 7. Cloning and working with repos (push/pull/branching)        │
│ ├── 8. Browsing code in the GCP Console                            │
│ ├── 9. Cross-repo code search                                      │
│ ├── 10. Integration with Cloud Build (CI/CD triggers)              │
│ ├── 11. Integration with other GCP services                        │
│ ├── 12. IAM roles and security                                     │
│ ├── 13. Pub/Sub notifications on repo events                       │
│ ├── 14. Terraform and gcloud CLI                                   │
│ └── 15. Real-world architecture patterns                           │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

> **Note:** Google announced CSR is deprecated as of June 2024 and will be shut down in June 2025. For new projects, use GitHub, GitLab, or Bitbucket with Cloud Build's direct connectors. This chapter is still relevant for understanding existing infrastructure, exam preparation, and the mirroring/integration concepts that carry over to other tools.

---

## Part 1: Cloud Source Repositories Fundamentals

```
┌─────────────────────────────────────────────────────────────────────┐
│           WHAT IS CLOUD SOURCE REPOSITORIES?                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ CSR = Fully managed, private Git hosting inside GCP                │
│       + automatic mirroring from GitHub/Bitbucket                  │
│       + native GCP service integrations                            │
│                                                                       │
│ Key characteristics:                                                │
│ ├── Standard Git — any Git client works (git clone, push, pull)    │
│ ├── Private by default — accessible only to project IAM members   │
│ ├── Up to 5 free repos per billing account (then $1/repo/month)    │
│ ├── Unlimited branches, tags, and Git history                      │
│ ├── Mirror mode — auto-sync from GitHub or Bitbucket              │
│ ├── Built-in code browser in GCP Console                           │
│ ├── Cross-repo code search (regex search across all repos)         │
│ ├── Pub/Sub integration (notifications on push/create)             │
│ ├── Native trigger for Cloud Build, Cloud Functions, App Engine    │
│ └── Scoped to a GCP project (one project = many repos)             │
│                                                                       │
│ Two modes of operation:                                             │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │                                                               │   │
│ │  1. HOSTED REPO (GCP-native)                                  │   │
│ │     ├── Git repo lives entirely in GCP                        │   │
│ │     ├── Push/pull directly to CSR                             │   │
│ │     ├── CSR is the source of truth                            │   │
│ │     └── Like a private GitHub repo but inside GCP             │   │
│ │                                                               │   │
│ │  2. MIRRORED REPO (sync from external)                        │   │
│ │     ├── Primary repo lives on GitHub or Bitbucket             │   │
│ │     ├── CSR creates a read-only mirror                        │   │
│ │     ├── Auto-syncs on every push to the source                │   │
│ │     ├── Mirror is read-only in CSR (push to GitHub instead)  │   │
│ │     └── Gives you GCP-native integrations without moving code│   │
│ │                                                               │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ When to use CSR:                                                    │
│ ├── You want source code browsable inside GCP Console             │
│ ├── You need tight Cloud Build / Cloud Functions triggers          │
│ ├── Your org mandates all resources within GCP                     │
│ ├── You want to search code across all GCP project repos           │
│ └── You mirror GitHub repos to integrate with GCP natively         │
│                                                                       │
│ When NOT to use CSR:                                                │
│ ├── You need PR reviews, issue tracking, wikis → GitHub/GitLab    │
│ ├── New projects (CSR is deprecated) → use GitHub + Cloud Build   │
│ ├── Large open-source collaboration → GitHub                       │
│ ├── Advanced CI/CD UI → GitLab, GitHub Actions                    │
│ └── You need branch protection, code owners → GitHub/GitLab       │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Cross-Cloud Comparison

```
┌──────────────────────┬───────────────┬───────────────┬───────────────┐
│ Feature              │ GCP CSR       │ AWS CodeCommit│ Azure Repos   │
├──────────────────────┼───────────────┼───────────────┼───────────────┤
│ Type                 │ Managed Git   │ Managed Git   │ Managed Git   │
│ Status               │ ⚠ Deprecated  │ ⚠ Deprecated  │ ✅ Active      │
│ Max repo size        │ 500 MB (soft) │ No hard limit │ 250 GB        │
│ Mirroring            │ ✅ GitHub +    │ ❌ No          │ ❌ No (import  │
│                      │ Bitbucket     │               │ only)         │
│ PR / Code Review     │ ❌ No          │ ✅ Basic       │ ✅ Full        │
│ Issue tracking       │ ❌ No          │ ❌ No          │ ✅ Azure Boards│
│ CI/CD integration    │ Cloud Build   │ CodePipeline  │ Azure Pipeline│
│ Code search          │ ✅ Cross-repo  │ ❌ No          │ ✅ Semantic    │
│ Pub/Sub on push      │ ✅ Yes         │ ✅ SNS/Lambda  │ ✅ Webhooks    │
│ IAM integration      │ ✅ GCP IAM     │ ✅ AWS IAM     │ ✅ Azure AD    │
│ Pricing              │ 5 free repos, │ 5 free users, │ 5 free users, │
│                      │ $1/repo after │ free repos    │ free repos    │
│ Recommended          │ Use GitHub +  │ Use GitHub +  │ Azure Repos   │
│ alternative          │ Cloud Build   │ CodePipeline  │ or GitHub     │
└──────────────────────┴───────────────┴───────────────┴───────────────┘

Note: Both GCP CSR and AWS CodeCommit are deprecated. The industry has
consolidated around GitHub, GitLab, and Bitbucket as source control
platforms, with cloud-native CI/CD services connecting to them directly.
```

### Pricing Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│           CSR PRICING                                                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ ┌────────────────────┬─────────────────────────────────────────┐   │
│ │ Component          │ Price                                    │   │
│ ├────────────────────┼─────────────────────────────────────────┤   │
│ │ First 5 project    │ FREE                                    │   │
│ │ users per billing  │                                         │   │
│ │ account            │                                         │   │
│ │                    │                                         │   │
│ │ Additional users   │ $1/user/month                           │   │
│ │                    │                                         │   │
│ │ Storage            │ First 50 GB free,                      │   │
│ │                    │ then $0.10/GB/month                     │   │
│ │                    │                                         │   │
│ │ Egress             │ First 50 GB free,                      │   │
│ │                    │ then $0.10/GB                           │   │
│ └────────────────────┴─────────────────────────────────────────┘   │
│                                                                       │
│ ⚡ Very cheap — most small/medium teams fall within free tier.     │
│ ⚡ Mirrored repos count as repos for user limits.                  │
│ ⚡ No per-repo charges (user-based pricing).                       │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 2: Architecture & How It Works

```
┌─────────────────────────────────────────────────────────────────────┐
│           CSR ARCHITECTURE                                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│                       GCP PROJECT                                   │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │                                                               │  │
│  │  Cloud Source Repositories                                   │  │
│  │                                                               │  │
│  │  ┌──────────────────┐  ┌──────────────────┐                 │  │
│  │  │ HOSTED REPO      │  │ HOSTED REPO      │                 │  │
│  │  │ "backend-api"    │  │ "frontend-app"   │                 │  │
│  │  │ (read/write)     │  │ (read/write)     │                 │  │
│  │  │                  │  │                  │                 │  │
│  │  │ main             │  │ main             │                 │  │
│  │  │ develop          │  │ develop          │                 │  │
│  │  │ feature/xyz      │  │ release/v2       │                 │  │
│  │  └──────────────────┘  └──────────────────┘                 │  │
│  │                                                               │  │
│  │  ┌──────────────────┐  ┌──────────────────┐                 │  │
│  │  │ MIRRORED REPO    │  │ MIRRORED REPO    │                 │  │
│  │  │ "github_org_     │  │ "bitbucket_team_ │                 │  │
│  │  │  mobile-app"     │  │  infra-code"     │                 │  │
│  │  │ (read-only)      │  │ (read-only)      │                 │  │
│  │  │                  │  │                  │                 │  │
│  │  │ ← syncs from     │  │ ← syncs from     │                 │  │
│  │  │   GitHub          │  │   Bitbucket      │                 │  │
│  │  └────────┬─────────┘  └────────┬─────────┘                 │  │
│  │           │                     │                            │  │
│  └───────────┼─────────────────────┼────────────────────────────┘  │
│              │                     │                               │
│              ▼                     ▼                               │
│  ┌──────────────────┐  ┌──────────────────┐                      │
│  │ GitHub           │  │ Bitbucket        │                      │
│  │ (source of truth)│  │ (source of truth)│                      │
│  └──────────────────┘  └──────────────────┘                      │
│                                                                       │
│ Integration points:                                                │
│                                                                       │
│  ┌──────────────────────────────────────────┐                      │
│  │        Cloud Source Repositories          │                      │
│  │                                           │                      │
│  │  repo push ──→ Pub/Sub notification      │                      │
│  │  repo push ──→ Cloud Build trigger       │                      │
│  │  repo push ──→ Cloud Functions deploy    │                      │
│  │  repo push ──→ App Engine deploy         │                      │
│  │  code ────────→ Console code browser     │                      │
│  │  code ────────→ Cross-repo search        │                      │
│  └──────────────────────────────────────────┘                      │
│                                                                       │
│ Key architectural facts:                                            │
│ ├── Repos are scoped to a GCP project                              │
│ ├── Repo names must be unique within a project                     │
│ ├── Standard Git protocol — any Git client works                   │
│ ├── Data stored in Google infrastructure (encrypted at rest)       │
│ ├── Mirrors sync automatically (typically within minutes)          │
│ ├── No webhooks from CSR — use Pub/Sub for event notifications    │
│ └── CMEK supported for repo encryption                             │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 3: Creating a Repository

### Console Walkthrough

```
┌─────────────────────────────────────────────────────────────────────┐
│ CONSOLE → Source Repositories → Add Repository                       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Step 1: Choose Repository Type                                     │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ ○ Create new repository                                       │   │
│ │   → Empty Git repo hosted in GCP                             │   │
│ │   → You push code directly to this repo                      │   │
│ │                                                               │   │
│ │ ○ Connect external repository                                 │   │
│ │   → Mirror from GitHub or Bitbucket                          │   │
│ │   → Read-only mirror in CSR                                  │   │
│ │   → (Covered in Parts 4 & 5)                                 │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ Step 2: Configure New Repository                                   │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ Repository name:     [backend-api]                            │   │
│ │   ├── Must be unique within the project                      │   │
│ │   ├── Allowed: letters, numbers, hyphens, underscores        │   │
│ │   └── Cannot start with a dot or end with .git               │   │
│ │                                                               │   │
│ │ Project:             [my-gcp-project]                         │   │
│ │   └── Repo belongs to this project (cannot move later)       │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ Step 3: After Creation — Push Code                                 │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ Console shows you the clone URL and setup instructions:      │   │
│ │                                                               │   │
│ │ # Option A: Clone empty repo, add code                       │   │
│ │ gcloud source repos clone backend-api --project=my-gcp-proj  │   │
│ │ cd backend-api                                                │   │
│ │ # ... add files ...                                           │   │
│ │ git add .                                                     │   │
│ │ git commit -m "Initial commit"                                │   │
│ │ git push -u origin main                                       │   │
│ │                                                               │   │
│ │ # Option B: Push existing local repo                         │   │
│ │ git remote add google \                                       │   │
│ │   https://source.developers.google.com/p/my-gcp-proj/\      │   │
│ │   r/backend-api                                               │   │
│ │ git push google main                                          │   │
│ │                                                               │   │
│ │ # Option C: Mirror from GitHub (see Part 4)                  │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ ⚡ Repo is created instantly (no provisioning delay).              │
│ ⚡ Empty until you push code.                                     │
│ ⚡ Default branch name follows your local Git config.              │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 4: Mirroring from GitHub

```
┌─────────────────────────────────────────────────────────────────────┐
│           GITHUB MIRRORING                                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ How it works:                                                       │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │                                                               │   │
│ │  Developer pushes to GitHub                                   │   │
│ │       │                                                       │   │
│ │       ▼                                                       │   │
│ │  ┌──────────────────┐                                        │   │
│ │  │ GitHub repo      │                                        │   │
│ │  │ (source of truth)│                                        │   │
│ │  └────────┬─────────┘                                        │   │
│ │           │ automatic sync (webhook)                         │   │
│ │           ▼                                                   │   │
│ │  ┌──────────────────┐                                        │   │
│ │  │ CSR mirror repo  │                                        │   │
│ │  │ (read-only copy) │                                        │   │
│ │  └────────┬─────────┘                                        │   │
│ │           │                                                   │   │
│ │    ┌──────┼──────┬──────────────┐                            │   │
│ │    ▼      ▼      ▼              ▼                            │   │
│ │ Cloud   Cloud    Console     Pub/Sub                         │   │
│ │ Build   Func.    browser     notification                    │   │
│ │ trigger deploy   & search                                    │   │
│ │                                                               │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ Setup process (Console):                                           │
│ 1. Console → Source Repositories → Add Repository                 │
│ 2. Select "Connect external repository"                           │
│ 3. Select project                                                  │
│ 4. Select "GitHub" as the Git provider                             │
│                                                                       │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ Step 4a: Authorize Cloud Source Repositories                  │   │
│ │ ├── Click "Connect to GitHub"                                │   │
│ │ ├── OAuth flow opens GitHub in browser                       │   │
│ │ ├── Authorize "Google Cloud Source Repositories" app         │   │
│ │ ├── Grant access to the specific repos or organizations      │   │
│ │ └── ⚠️ Uses a personal GitHub OAuth token                    │   │
│ │                                                               │   │
│ │ Step 4b: Select repository to mirror                         │   │
│ │ ├── Drop-down shows all accessible GitHub repos              │   │
│ │ ├── Select repo: [my-org/mobile-app]                         │   │
│ │ ├── ☑ Automatically mirror changes from GitHub               │   │
│ │ └── Click "Connect selected repository"                      │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ Result:                                                             │
│ ├── Mirror repo created: "github_my-org_mobile-app"                │
│ │   (naming convention: github_{owner}_{repo})                     │
│ ├── All branches and tags synced automatically                     │
│ ├── Sync happens within minutes of any push to GitHub              │
│ ├── Mirror is read-only — cannot push to CSR copy                 │
│ └── If GitHub repo is deleted, mirror becomes stale                │
│                                                                       │
│ ⚠️ Important notes:                                                │
│ ├── Requires GitHub OAuth authorization (personal account)         │
│ ├── The authorizing user must have read access to the GitHub repo │
│ ├── If the user's GitHub access is revoked, mirror stops syncing  │
│ ├── Large repos (> 500 MB) may have sync issues                   │
│ └── GitHub Enterprise Server: may require additional networking    │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 5: Mirroring from Bitbucket

```
┌─────────────────────────────────────────────────────────────────────┐
│           BITBUCKET MIRRORING                                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Same concept as GitHub mirroring but for Bitbucket Cloud.          │
│                                                                       │
│ Setup process:                                                      │
│ 1. Console → Source Repositories → Add Repository                 │
│ 2. Select "Connect external repository"                           │
│ 3. Select "Bitbucket" as the Git provider                          │
│                                                                       │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ Step 3a: Authorize Bitbucket                                  │   │
│ │ ├── Click "Connect to Bitbucket"                             │   │
│ │ ├── OAuth flow opens Bitbucket in browser                    │   │
│ │ ├── Grant access to Google Cloud Source Repositories          │   │
│ │ └── Uses Bitbucket OAuth consumer                            │   │
│ │                                                               │   │
│ │ Step 3b: Select repository                                   │   │
│ │ ├── Drop-down shows accessible Bitbucket repos               │   │
│ │ ├── Select repo: [my-team/infra-code]                        │   │
│ │ ├── ☑ Automatically mirror changes                           │   │
│ │ └── Click "Connect selected repository"                      │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ Result:                                                             │
│ ├── Mirror repo: "bitbucket_my-team_infra-code"                    │
│ ├── Naming: bitbucket_{workspace}_{repo}                           │
│ ├── Same behavior as GitHub mirror (read-only, auto-sync)          │
│ └── All branches, tags, and history mirrored                       │
│                                                                       │
│ Differences from GitHub mirroring:                                 │
│ ├── Different OAuth provider but same UX in GCP Console            │
│ ├── Only Bitbucket Cloud supported (not Bitbucket Server)          │
│ ├── App password can also be used for authentication               │
│ └── Same sync frequency and limitations                            │
│                                                                       │
│ Troubleshooting sync issues:                                       │
│ ├── Check that authorizing user still has repo access              │
│ ├── Re-authorize OAuth if token expired                            │
│ ├── Verify webhook is active on GitHub/Bitbucket side             │
│ ├── Check Cloud Logging for sync errors                            │
│ └── Manual sync: delete and re-create the mirror connection        │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 6: Authentication & Access

```
┌─────────────────────────────────────────────────────────────────────┐
│           AUTHENTICATION METHODS                                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ 1. GCLOUD CREDENTIAL HELPER (recommended)                          │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ Uses your gcloud auth credentials automatically.             │   │
│ │                                                               │   │
│ │ # One-time setup: configure Git to use gcloud for auth       │   │
│ │ git config --global credential.https://source.developers.\   │   │
│ │   google.com.helper gcloud.sh                                 │   │
│ │                                                               │   │
│ │ # Now clone works automatically                              │   │
│ │ gcloud source repos clone backend-api --project=my-gcp-proj  │   │
│ │                                                               │   │
│ │ # Or use the HTTPS URL directly                              │   │
│ │ git clone https://source.developers.google.com/p/\           │   │
│ │   my-gcp-proj/r/backend-api                                   │   │
│ │                                                               │   │
│ │ ✅ Simplest method                                            │   │
│ │ ✅ Uses existing gcloud authentication                        │   │
│ │ ✅ Supports MFA / org policies                                │   │
│ │ ⚠️ Requires gcloud SDK installed                              │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ 2. SSH KEY                                                         │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ # Generate SSH key (if you don't have one)                    │   │
│ │ ssh-keygen -t ed25519 -C "your-email@example.com"             │   │
│ │                                                               │   │
│ │ # Register SSH key with Cloud Source Repositories             │   │
│ │ # Console → Source Repositories → Manage SSH Keys            │   │
│ │ # Or: Console → your user icon → SSH Keys                   │   │
│ │ # Paste the contents of ~/.ssh/id_ed25519.pub                │   │
│ │                                                               │   │
│ │ # Clone via SSH                                               │   │
│ │ git clone ssh://your-email@source.developers.google.com:2022/│   │
│ │   p/my-gcp-proj/r/backend-api                                │   │
│ │                                                               │   │
│ │ ✅ Familiar SSH-based workflow                                │   │
│ │ ✅ No gcloud SDK required                                     │   │
│ │ ⚠️ SSH key must be registered per user                        │   │
│ │ ⚠️ Port 2022 (not standard 22)                                │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ 3. MANUALLY GENERATED CREDENTIALS                                  │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ For environments where gcloud SDK is not available:          │   │
│ │                                                               │   │
│ │ # Generate static credentials                                │   │
│ │ # Console → Source Repositories → Clone → Manually           │   │
│ │ #   generated credentials                                    │   │
│ │ # Generates a password to use with HTTPS Git                 │   │
│ │                                                               │   │
│ │ git clone https://source.developers.google.com/p/\           │   │
│ │   my-gcp-proj/r/backend-api                                   │   │
│ │ # Username: your-email@example.com                           │   │
│ │ # Password: (generated credential)                           │   │
│ │                                                               │   │
│ │ ✅ Works in any environment                                   │   │
│ │ ⚠️ Static credential — should be stored securely             │   │
│ │ ⚠️ Can be revoked in Console                                 │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ 4. SERVICE ACCOUNT (for CI/CD or automation)                       │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ # Activate service account                                    │   │
│ │ gcloud auth activate-service-account \                        │   │
│ │   --key-file=sa-key.json                                      │   │
│ │                                                               │   │
│ │ # Configure credential helper                                │   │
│ │ git config credential.helper gcloud.sh                        │   │
│ │                                                               │   │
│ │ # Clone as the service account                                │   │
│ │ gcloud source repos clone backend-api --project=my-gcp-proj  │   │
│ │                                                               │   │
│ │ ✅ For CI/CD pipelines, Cloud Build, automation              │   │
│ │ ✅ No human credentials needed                                │   │
│ │ ⚠️ Service account needs source.repos.reader/writer role     │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 7: Cloning & Working with Repos

```
┌─────────────────────────────────────────────────────────────────────┐
│           WORKING WITH CSR REPOS                                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ It's standard Git — everything you know about Git applies.         │
│                                                                       │
│ Clone a repo:                                                       │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ # Using gcloud helper (recommended)                           │   │
│ │ gcloud source repos clone backend-api --project=my-gcp-proj  │   │
│ │                                                               │   │
│ │ # Using HTTPS URL                                             │   │
│ │ git clone https://source.developers.google.com/p/\           │   │
│ │   my-gcp-proj/r/backend-api                                   │   │
│ │                                                               │   │
│ │ # Using SSH                                                   │   │
│ │ git clone ssh://user@source.developers.google.com:2022/\     │   │
│ │   p/my-gcp-proj/r/backend-api                                │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ Daily workflow (same as any Git repo):                              │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ # Create and switch to a branch                               │   │
│ │ git checkout -b feature/add-auth                              │   │
│ │                                                               │   │
│ │ # Make changes, stage, commit                                 │   │
│ │ vim src/auth.py                                               │   │
│ │ git add src/auth.py                                           │   │
│ │ git commit -m "Add OAuth2 authentication"                     │   │
│ │                                                               │   │
│ │ # Push to CSR                                                 │   │
│ │ git push origin feature/add-auth                              │   │
│ │                                                               │   │
│ │ # Pull latest from main                                       │   │
│ │ git checkout main                                             │   │
│ │ git pull origin main                                          │   │
│ │                                                               │   │
│ │ # Tags                                                        │   │
│ │ git tag v1.0.0                                                │   │
│ │ git push origin v1.0.0                                        │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ Multi-remote workflow (push to both GitHub AND CSR):               │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ # Your repo already has GitHub as "origin"                    │   │
│ │ git remote -v                                                 │   │
│ │ # origin  https://github.com/my-org/backend-api.git (fetch)  │   │
│ │ # origin  https://github.com/my-org/backend-api.git (push)   │   │
│ │                                                               │   │
│ │ # Add CSR as a second remote                                  │   │
│ │ git remote add google \                                       │   │
│ │   https://source.developers.google.com/p/my-gcp-proj/\      │   │
│ │   r/backend-api                                               │   │
│ │                                                               │   │
│ │ # Push to both                                                │   │
│ │ git push origin main   # → GitHub                            │   │
│ │ git push google main   # → CSR                               │   │
│ │                                                               │   │
│ │ ⚠️ For mirrored repos, this is handled automatically.        │   │
│ │    Multi-remote is only for hosted repos.                    │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ ⚠️ No pull requests in CSR — it's just Git hosting.               │
│    Use GitHub/GitLab for code review, then push/mirror to CSR.    │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 8: Browsing Code in Console

```
┌─────────────────────────────────────────────────────────────────────┐
│           CONSOLE CODE BROWSER                                        │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Console → Source Repositories → [Select repo]                     │
│                                                                       │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │                                                               │   │
│ │  Branch/Tag selector: [main ▼]  |  Path: /src/               │   │
│ │                                                               │   │
│ │  ┌────────────────────────────────────────────────────────┐  │   │
│ │  │ 📁 src/                                                │  │   │
│ │  │   📁 controllers/                                      │  │   │
│ │  │     📄 auth.py                  2 hours ago             │  │   │
│ │  │     📄 users.py                 3 days ago              │  │   │
│ │  │   📁 models/                                            │  │   │
│ │  │     📄 user.py                  1 week ago              │  │   │
│ │  │   📄 main.py                    5 hours ago             │  │   │
│ │  │ 📄 Dockerfile                   2 weeks ago             │  │   │
│ │  │ 📄 requirements.txt             2 weeks ago             │  │   │
│ │  │ 📄 README.md                    1 month ago             │  │   │
│ │  └────────────────────────────────────────────────────────┘  │   │
│ │                                                               │   │
│ │  Click a file to view:                                       │   │
│ │  ┌────────────────────────────────────────────────────────┐  │   │
│ │  │ src/main.py                   [Blame] [History] [Raw]  │  │   │
│ │  │ ────────────────────────────────────────────────────── │  │   │
│ │  │  1 │ from flask import Flask                           │  │   │
│ │  │  2 │ from controllers import auth, users              │  │   │
│ │  │  3 │                                                   │  │   │
│ │  │  4 │ app = Flask(__name__)                             │  │   │
│ │  │  5 │ app.register_blueprint(auth.bp)                  │  │   │
│ │  │  6 │ app.register_blueprint(users.bp)                 │  │   │
│ │  │  7 │                                                   │  │   │
│ │  │  8 │ if __name__ == "__main__":                        │  │   │
│ │  │  9 │     app.run(host="0.0.0.0", port=8080)           │  │   │
│ │  └────────────────────────────────────────────────────────┘  │   │
│ │                                                               │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ Features:                                                           │
│ ├── Browse any branch or tag                                       │
│ ├── Syntax highlighting for common languages                       │
│ ├── File blame (git blame — who changed each line)                │
│ ├── Commit history per file                                        │
│ ├── Raw file download                                              │
│ ├── Directory tree navigation                                      │
│ └── Last commit message and timestamp per file                     │
│                                                                       │
│ ⚡ Useful for quick code checks without cloning locally.           │
│ ⚡ Works for both hosted and mirrored repos.                       │
│ ⚡ Not a full IDE — use Cloud Shell Editor or local IDE for edits.│
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 9: Search Across Repositories

```
┌─────────────────────────────────────────────────────────────────────┐
│           CROSS-REPO CODE SEARCH                                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Console → Source Repositories → Search (top bar)                   │
│                                                                       │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ Search: [database_url                    ] [🔍]              │   │
│ │                                                               │   │
│ │ Filters:                                                      │   │
│ │ ├── Repository: [All repos ▼]                                │   │
│ │ ├── File path: [*.py]     (glob patterns supported)          │   │
│ │ ├── Language: [Python ▼]                                     │   │
│ │ └── Regular expression: ☑                                    │   │
│ │                                                               │   │
│ │ Results (3 matches across 2 repos):                          │   │
│ │                                                               │   │
│ │  📦 backend-api / src/config.py:15                           │   │
│ │  │   DATABASE_URL = os.getenv("DATABASE_URL")                │   │
│ │  │                                                           │   │
│ │  📦 backend-api / src/models/base.py:8                       │   │
│ │  │   engine = create_engine(settings.DATABASE_URL)            │   │
│ │  │                                                           │   │
│ │  📦 worker-service / src/db.py:12                            │   │
│ │  │   conn_string = DATABASE_URL or "postgresql://..."        │   │
│ │                                                               │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ Search capabilities:                                                │
│ ├── Plain text search (case-insensitive by default)                │
│ ├── Regular expression support                                     │
│ ├── Filter by file path pattern (glob)                             │
│ ├── Filter by language                                             │
│ ├── Filter by specific repository or search all repos              │
│ ├── Shows file path, line number, and code context                 │
│ ├── Click result to jump to file viewer                            │
│ └── Searches across ALL repos in the project (hosted + mirrored)  │
│                                                                       │
│ Example search patterns:                                           │
│ ├── "TODO" — find all TODOs across all repos                      │
│ ├── "apikey|api_key|API_KEY" (regex) — find API key references    │
│ ├── "password" in *.yaml — find passwords in YAML files           │
│ └── "func.*Handler" (regex) — find Go handler functions           │
│                                                                       │
│ ⚡ Useful for auditing, refactoring, finding usage patterns.      │
│ ⚡ Only searches the default branch (usually main/master).        │
│ ⚡ Limited to repos within the current GCP project.               │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 10: Integration with Cloud Build

```
┌─────────────────────────────────────────────────────────────────────┐
│           CSR + CLOUD BUILD INTEGRATION                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ The primary reason to use CSR: native Cloud Build triggers.        │
│                                                                       │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │                                                               │   │
│ │  Developer pushes code                                        │   │
│ │       │                                                       │   │
│ │       ▼                                                       │   │
│ │  ┌──────────────────┐     ┌──────────────────────────────┐   │   │
│ │  │ Cloud Source      │────→│ Cloud Build Trigger           │   │   │
│ │  │ Repositories      │     │                               │   │   │
│ │  │                  │     │ Event: Push to branch "main"  │   │   │
│ │  │ "backend-api"    │     │ File: cloudbuild.yaml          │   │   │
│ │  └──────────────────┘     └──────────────┬─────────────────┘   │   │
│ │                                           │                    │   │
│ │                                           ▼                    │   │
│ │                            ┌─────────────────────────────┐    │   │
│ │                            │ Cloud Build                  │    │   │
│ │                            │                              │    │   │
│ │                            │ Step 1: Build Docker image   │    │   │
│ │                            │ Step 2: Run tests            │    │   │
│ │                            │ Step 3: Push to Artifact Reg.│    │   │
│ │                            │ Step 4: Deploy to Cloud Run  │    │   │
│ │                            └─────────────────────────────┘    │   │
│ │                                                               │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ Creating a Cloud Build trigger for CSR:                            │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ Console → Cloud Build → Triggers → Create Trigger           │   │
│ │                                                               │   │
│ │ Name:            [deploy-on-push]                             │   │
│ │ Event:           [Push to a branch]                           │   │
│ │ Source:          [Cloud Source Repositories]                  │   │
│ │ Repository:      [backend-api]                                │   │
│ │ Branch:          [^main$]   (regex)                          │   │
│ │                                                               │   │
│ │ Build config:    [Cloud Build configuration file]            │   │
│ │ File location:   [/cloudbuild.yaml]                          │   │
│ │                                                               │   │
│ │ Included files:  [src/**] (optional — only trigger on src)   │   │
│ │ Ignored files:   [docs/**] (optional — skip docs changes)    │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ Trigger events:                                                     │
│ ├── Push to a branch (regex match on branch name)                  │
│ ├── Push new tag (regex match on tag name)                         │
│ └── No PR trigger (CSR has no pull requests)                       │
│                                                                       │
│ For mirrored repos:                                                │
│ ├── Push to GitHub → mirror syncs → Cloud Build trigger fires    │
│ ├── Works exactly the same as hosted repos                        │
│ └── Small delay: GitHub push → mirror sync → trigger (seconds)   │
│                                                                       │
│ ⚡ With CSR deprecated, use Cloud Build's native GitHub/Bitbucket │
│    connectors instead (2nd gen triggers connect directly to       │
│    GitHub without needing CSR as an intermediary).                │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 11: Integration with Other GCP Services

```
┌─────────────────────────────────────────────────────────────────────┐
│           CSR INTEGRATIONS                                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ 1. CLOUD FUNCTIONS                                                 │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ Deploy a Cloud Function directly from a CSR repo:            │   │
│ │                                                               │   │
│ │ gcloud functions deploy my-function \                         │   │
│ │   --source=https://source.developers.google.com/projects/\   │   │
│ │     my-gcp-proj/repos/backend-api/moveable-aliases/main/\   │   │
│ │     paths/functions/hello-world \                             │   │
│ │   --trigger-http \                                            │   │
│ │   --runtime=python311                                         │   │
│ │                                                               │   │
│ │ Console → Cloud Functions → Create → Source Code              │   │
│ │   → Cloud Source Repository → Select repo & branch/tag      │   │
│ │   → Specify directory containing function code               │   │
│ │                                                               │   │
│ │ ✅ Deploy from specific branch, tag, or commit               │   │
│ │ ✅ Auto-deploy by combining with Cloud Build trigger         │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ 2. APP ENGINE                                                      │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ Deploy App Engine app from CSR:                               │   │
│ │                                                               │   │
│ │ # Clone from CSR and deploy                                   │   │
│ │ gcloud source repos clone frontend-app                        │   │
│ │ cd frontend-app                                               │   │
│ │ gcloud app deploy                                             │   │
│ │                                                               │   │
│ │ Or automate with Cloud Build trigger on push.                │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ 3. CLOUD SHELL                                                     │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ Open any CSR repo directly in Cloud Shell:                   │   │
│ │                                                               │   │
│ │ Console → Source Repositories → [repo] →                    │   │
│ │   "Open in Cloud Shell" button                               │   │
│ │                                                               │   │
│ │ ├── Clones the repo into your Cloud Shell environment        │   │
│ │ ├── Opens Cloud Shell Editor (VS Code-like)                  │   │
│ │ ├── Ready to edit, build, and test                           │   │
│ │ └── Changes can be pushed back to CSR                        │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ 4. DEBUGGER (Cloud Debugger — also deprecated)                     │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ Cloud Debugger could attach to running code using CSR as the │   │
│ │ source for breakpoint navigation.                             │   │
│ │                                                               │   │
│ │ ⚠️ Cloud Debugger is deprecated alongside CSR.               │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ 5. ERROR REPORTING                                                 │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ Error Reporting can link stack traces to source code in CSR. │   │
│ │ Click a stack frame → opens the exact file/line in CSR.     │   │
│ │                                                               │   │
│ │ Requires:                                                     │   │
│ │ ├── Source code in CSR (hosted or mirrored)                  │   │
│ │ ├── Application deployed with matching source version        │   │
│ │ └── Works with Cloud Functions, App Engine, GKE              │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 12: IAM & Security

```
┌─────────────────────────────────────────────────────────────────────┐
│           IAM ROLES FOR CLOUD SOURCE REPOSITORIES                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Predefined roles:                                                   │
│ ┌──────────────────────────────┬──────────────────────────────────┐│
│ │ Role                         │ Permissions                       ││
│ ├──────────────────────────────┼──────────────────────────────────┤│
│ │ source.reader                │ Clone and read repos              ││
│ │                              │ Browse code in Console            ││
│ │                              │ Search code                       ││
│ │                              │                                   ││
│ │ source.writer                │ All reader permissions +          ││
│ │                              │ Push to repos (git push)          ││
│ │                              │ Create branches and tags          ││
│ │                              │                                   ││
│ │ source.admin                 │ All writer permissions +          ││
│ │                              │ Create and delete repos           ││
│ │                              │ Configure mirroring               ││
│ │                              │ Manage repo settings              ││
│ └──────────────────────────────┴──────────────────────────────────┘│
│                                                                       │
│ Granting access:                                                    │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ # Grant read access to a user                                │   │
│ │ gcloud projects add-iam-policy-binding my-gcp-proj \          │   │
│ │   --member="user:dev@example.com" \                           │   │
│ │   --role="roles/source.reader"                                │   │
│ │                                                               │   │
│ │ # Grant write access to a service account (for CI/CD)         │   │
│ │ gcloud projects add-iam-policy-binding my-gcp-proj \          │   │
│ │   --member="serviceAccount:builder@my-proj.iam.gserviceaccount│   │
│ │     .com" \                                                   │   │
│ │   --role="roles/source.writer"                                │   │
│ │                                                               │   │
│ │ # Grant admin access to a group                               │   │
│ │ gcloud projects add-iam-policy-binding my-gcp-proj \          │   │
│ │   --member="group:platform-team@example.com" \                │   │
│ │   --role="roles/source.admin"                                 │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ ⚠️ IAM is project-level by default. All repos in a project share  │
│    the same IAM policies. You cannot restrict access to a single  │
│    repo unless you put it in a separate project.                  │
│                                                                       │
│ Security features:                                                  │
│ ├── Encryption at rest: Google-managed keys (default) or CMEK     │
│ ├── Encryption in transit: TLS (HTTPS or SSH)                     │
│ ├── Audit logging: Admin Activity (auto), Data Access (opt-in)    │
│ ├── VPC Service Controls: Restrict CSR access to service perimeter│
│ ├── Organization policies: Restrict repo creation to specific     │
│ │   projects                                                       │
│ └── Data residency: Code stored in Google's US infrastructure     │
│     (no region selection for CSR)                                  │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 13: Notifications & Pub/Sub

```
┌─────────────────────────────────────────────────────────────────────┐
│           PUB/SUB NOTIFICATIONS                                       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ CSR can publish events to Pub/Sub when repo changes occur.         │
│ This enables event-driven workflows beyond Cloud Build triggers.   │
│                                                                       │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │                                                               │   │
│ │  git push                                                     │   │
│ │       │                                                       │   │
│ │       ▼                                                       │   │
│ │  ┌──────────────────┐                                        │   │
│ │  │ CSR repo          │                                        │   │
│ │  │ "backend-api"     │                                        │   │
│ │  └────────┬──────────┘                                        │   │
│ │           │ notification                                      │   │
│ │           ▼                                                   │   │
│ │  ┌──────────────────┐                                        │   │
│ │  │ Pub/Sub topic     │                                        │   │
│ │  │ "repo-events"     │                                        │   │
│ │  └────────┬──────────┘                                        │   │
│ │           │                                                   │   │
│ │    ┌──────┼──────┬──────────┐                                │   │
│ │    ▼      ▼      ▼          ▼                                │   │
│ │ Cloud   Cloud  Custom     Slack                              │   │
│ │ Func.   Run    webhook    notification                       │   │
│ │ (lint)  (test) (Jira)     (via function)                     │   │
│ │                                                               │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ Enable Pub/Sub notifications:                                      │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ # Create a Pub/Sub topic                                      │   │
│ │ gcloud pubsub topics create repo-events                       │   │
│ │                                                               │   │
│ │ # Enable notifications for a repo                             │   │
│ │ gcloud source repos update backend-api \                      │   │
│ │   --add-topic=projects/my-gcp-proj/topics/repo-events \       │   │
│ │   --message-format=json                                       │   │
│ │                                                               │   │
│ │ # Or for specific events:                                    │   │
│ │ gcloud source repos update backend-api \                      │   │
│ │   --add-topic=projects/my-gcp-proj/topics/repo-events \       │   │
│ │   --service-account=project-SA@my-gcp-proj.iam.\             │   │
│ │     gserviceaccount.com \                                     │   │
│ │   --message-format=json                                       │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ Pub/Sub message payload (JSON):                                    │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ {                                                             │   │
│ │   "name": "projects/my-gcp-proj/repos/backend-api",          │   │
│ │   "url": "https://source.developers.google.com/...",          │   │
│ │   "eventTime": "2024-01-15T10:30:00Z",                       │   │
│ │   "refUpdateEvent": {                                         │   │
│ │     "email": "dev@example.com",                               │   │
│ │     "refUpdates": {                                           │   │
│ │       "refs/heads/main": {                                    │   │
│ │         "refName": "refs/heads/main",                         │   │
│ │         "updateType": "UPDATE_FAST_FORWARD",                  │   │
│ │         "oldId": "abc123...",                                  │   │
│ │         "newId": "def456..."                                  │   │
│ │       }                                                       │   │
│ │     }                                                         │   │
│ │   }                                                           │   │
│ │ }                                                             │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ Use cases for Pub/Sub notifications:                               │
│ ├── Trigger Slack/Teams notification on push                       │
│ ├── Run linters or security scans via Cloud Functions              │
│ ├── Update Jira tickets when specific branches are pushed          │
│ ├── Trigger custom deployment pipelines                            │
│ └── Audit trail of all repo activities                             │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 14: Terraform & CLI

### Terraform

```hcl
# ─────────────────────────────────────────────────────────────
# Hosted Repository
# ─────────────────────────────────────────────────────────────

resource "google_sourcerepo_repository" "backend_api" {
  name    = "backend-api"
  project = "my-gcp-project"
}

resource "google_sourcerepo_repository" "frontend_app" {
  name    = "frontend-app"
  project = "my-gcp-project"
}

# ─────────────────────────────────────────────────────────────
# Repository with Pub/Sub Notification
# ─────────────────────────────────────────────────────────────

resource "google_pubsub_topic" "repo_events" {
  name    = "repo-events"
  project = "my-gcp-project"
}

resource "google_sourcerepo_repository" "worker_service" {
  name    = "worker-service"
  project = "my-gcp-project"

  pubsub_configs {
    topic                 = google_pubsub_topic.repo_events.id
    message_format        = "JSON"
    service_account_email = google_service_account.repo_notifier.email
  }
}

# ─────────────────────────────────────────────────────────────
# Service Account for Pub/Sub Publishing
# ─────────────────────────────────────────────────────────────

resource "google_service_account" "repo_notifier" {
  account_id   = "repo-notifier"
  display_name = "Repo Event Notifier"
  project      = "my-gcp-project"
}

resource "google_pubsub_topic_iam_member" "publisher" {
  topic   = google_pubsub_topic.repo_events.name
  role    = "roles/pubsub.publisher"
  member  = "serviceAccount:${google_service_account.repo_notifier.email}"
  project = "my-gcp-project"
}

# ─────────────────────────────────────────────────────────────
# IAM — Grant developer read/write access
# ─────────────────────────────────────────────────────────────

resource "google_project_iam_member" "dev_source_writer" {
  project = "my-gcp-project"
  role    = "roles/source.writer"
  member  = "group:developers@example.com"
}

resource "google_project_iam_member" "cicd_source_reader" {
  project = "my-gcp-project"
  role    = "roles/source.reader"
  member  = "serviceAccount:cloud-build@my-gcp-project.iam.gserviceaccount.com"
}

# ─────────────────────────────────────────────────────────────
# Cloud Build Trigger from CSR
# ─────────────────────────────────────────────────────────────

resource "google_cloudbuild_trigger" "deploy_on_push" {
  name        = "deploy-on-push"
  description = "Deploy backend-api on push to main"
  project     = "my-gcp-project"

  trigger_template {
    repo_name   = google_sourcerepo_repository.backend_api.name
    branch_name = "^main$"
  }

  filename = "cloudbuild.yaml"

  included_files = ["src/**", "Dockerfile"]
  ignored_files  = ["docs/**", "*.md"]
}

resource "google_cloudbuild_trigger" "deploy_on_tag" {
  name        = "deploy-on-tag"
  description = "Deploy on version tag"
  project     = "my-gcp-project"

  trigger_template {
    repo_name = google_sourcerepo_repository.backend_api.name
    tag_name  = "^v\\d+\\.\\d+\\.\\d+$"
  }

  filename = "cloudbuild.yaml"

  substitutions = {
    _DEPLOY_ENV = "production"
  }
}
```

### gcloud CLI Reference

```bash
# ─────────────────────────────────────────────────────────────
# Repository Management
# ─────────────────────────────────────────────────────────────

# Create a new hosted repo
gcloud source repos create backend-api

# List all repos in current project
gcloud source repos list

# Describe a repo
gcloud source repos describe backend-api

# Delete a repo (⚠️ irreversible!)
gcloud source repos delete backend-api

# ─────────────────────────────────────────────────────────────
# Cloning
# ─────────────────────────────────────────────────────────────

# Clone a repo (sets up gcloud credential helper automatically)
gcloud source repos clone backend-api

# Clone a repo from a different project
gcloud source repos clone backend-api --project=other-project

# Clone into a specific directory
gcloud source repos clone backend-api /path/to/local/dir

# ─────────────────────────────────────────────────────────────
# Pub/Sub Notifications
# ─────────────────────────────────────────────────────────────

# Add Pub/Sub notification to a repo
gcloud source repos update backend-api \
  --add-topic=projects/my-gcp-proj/topics/repo-events \
  --message-format=json

# Remove Pub/Sub notification from a repo
gcloud source repos update backend-api \
  --remove-topic=projects/my-gcp-proj/topics/repo-events

# Update notification format
gcloud source repos update backend-api \
  --update-topic=projects/my-gcp-proj/topics/repo-events \
  --message-format=protobuf

# ─────────────────────────────────────────────────────────────
# IAM
# ─────────────────────────────────────────────────────────────

# Get IAM policy for a repo
gcloud source repos get-iam-policy backend-api

# Set IAM policy for a repo
gcloud source repos set-iam-policy backend-api policy.json

# Grant read access
gcloud projects add-iam-policy-binding my-gcp-proj \
  --member="user:dev@example.com" \
  --role="roles/source.reader"

# Grant write access to a group
gcloud projects add-iam-policy-binding my-gcp-proj \
  --member="group:devs@example.com" \
  --role="roles/source.writer"

# Grant admin access
gcloud projects add-iam-policy-binding my-gcp-proj \
  --member="user:admin@example.com" \
  --role="roles/source.admin"

# ─────────────────────────────────────────────────────────────
# Git Credential Helper Setup
# ─────────────────────────────────────────────────────────────

# Configure gcloud as Git credential helper (global)
git config --global credential.https://source.developers.google.com.helper gcloud.sh

# Verify configuration
git config --global --get credential.https://source.developers.google.com.helper

# ─────────────────────────────────────────────────────────────
# Mirroring (Console only — no gcloud command)
# ─────────────────────────────────────────────────────────────

# Mirroring is configured via Console only:
# Console → Source Repositories → Add Repository →
#   Connect external repository → GitHub / Bitbucket
#
# ⚠️ Cannot set up mirroring via gcloud CLI or Terraform.
#    Must be done in the Console with OAuth authorization.

# ─────────────────────────────────────────────────────────────
# Cloud Build Trigger (using CSR as source)
# ─────────────────────────────────────────────────────────────

# Create trigger for push to main
gcloud builds triggers create cloud-source-repositories \
  --name=deploy-on-push \
  --repo=backend-api \
  --branch-pattern="^main$" \
  --build-config=cloudbuild.yaml

# Create trigger for tags
gcloud builds triggers create cloud-source-repositories \
  --name=deploy-on-tag \
  --repo=backend-api \
  --tag-pattern="^v.*" \
  --build-config=cloudbuild.yaml

# List triggers
gcloud builds triggers list

# Delete trigger
gcloud builds triggers delete deploy-on-push
```

---

## Part 15: Real-World Patterns

```
┌─────────────────────────────────────────────────────────────────────┐
│           PATTERN 1: GITHUB MIRROR + CLOUD BUILD CI/CD                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Scenario: Team uses GitHub for code review / PRs but deploys      │
│ to GCP using Cloud Build.                                          │
│                                                                       │
│ Architecture:                                                       │
│ ┌────────────┐                                                     │
│ │ Developer  │                                                     │
│ │ (PR on     │                                                     │
│ │  GitHub)   │                                                     │
│ └──────┬─────┘                                                     │
│        │ push / merge                                              │
│        ▼                                                           │
│ ┌──────────────┐      auto        ┌──────────────────────┐        │
│ │ GitHub       │────────sync──────→│ CSR Mirror           │        │
│ │ my-org/      │                  │ "github_my-org_      │        │
│ │ backend-api  │                  │  backend-api"        │        │
│ └──────────────┘                  └──────────┬───────────┘        │
│                                              │ trigger            │
│                                              ▼                    │
│                                   ┌──────────────────────┐        │
│                                   │ Cloud Build           │        │
│                                   │ 1. Build image        │        │
│                                   │ 2. Run tests          │        │
│                                   │ 3. Push to Artifact   │        │
│                                   │    Registry           │        │
│                                   │ 4. Deploy to Cloud Run│        │
│                                   └──────────────────────┘        │
│                                                                       │
│ Why this pattern:                                                  │
│ ├── GitHub for collaboration (PRs, code review, issues)            │
│ ├── CSR mirror for GCP-native CI/CD triggers                       │
│ ├── No need to configure webhooks manually                        │
│ ├── Everything stays within GCP project for governance             │
│ └── Team doesn't change their GitHub workflow                      │
│                                                                       │
│ ⚠️ Modern alternative (recommended):                               │
│ Use Cloud Build 2nd-gen triggers that connect directly to GitHub  │
│ without needing CSR as an intermediary. No mirror needed.         │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│           PATTERN 2: MULTI-REPO MONOREPO WITH SELECTIVE TRIGGERS      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Scenario: Monorepo with multiple services; only build what changed.│
│                                                                       │
│ Repository structure:                                               │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ my-monorepo/                                                  │   │
│ │ ├── services/                                                 │   │
│ │ │   ├── api/                                                  │   │
│ │ │   │   ├── Dockerfile                                       │   │
│ │ │   │   ├── cloudbuild.yaml                                  │   │
│ │ │   │   └── src/                                              │   │
│ │ │   ├── worker/                                               │   │
│ │ │   │   ├── Dockerfile                                       │   │
│ │ │   │   ├── cloudbuild.yaml                                  │   │
│ │ │   │   └── src/                                              │   │
│ │ │   └── frontend/                                             │   │
│ │ │       ├── Dockerfile                                       │   │
│ │ │       ├── cloudbuild.yaml                                  │   │
│ │ │       └── src/                                              │   │
│ │ ├── shared/                                                   │   │
│ │ │   └── utils/                                                │   │
│ │ └── infrastructure/                                           │   │
│ │     └── terraform/                                            │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ Cloud Build triggers (one per service):                            │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ Trigger 1: "build-api"                                        │   │
│ │ ├── Branch: ^main$                                           │   │
│ │ ├── Included files: services/api/**                          │   │
│ │ ├── Build config: services/api/cloudbuild.yaml               │   │
│ │ └── Only fires when api/ code changes                        │   │
│ │                                                               │   │
│ │ Trigger 2: "build-worker"                                    │   │
│ │ ├── Branch: ^main$                                           │   │
│ │ ├── Included files: services/worker/**                       │   │
│ │ ├── Build config: services/worker/cloudbuild.yaml            │   │
│ │ └── Only fires when worker/ code changes                     │   │
│ │                                                               │   │
│ │ Trigger 3: "build-frontend"                                  │   │
│ │ ├── Branch: ^main$                                           │   │
│ │ ├── Included files: services/frontend/**                     │   │
│ │ └── Build config: services/frontend/cloudbuild.yaml          │   │
│ │                                                               │   │
│ │ Trigger 4: "build-all" (when shared code changes)            │   │
│ │ ├── Branch: ^main$                                           │   │
│ │ ├── Included files: shared/**                                │   │
│ │ └── Builds all services (shared dependency changed)          │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ ⚡ Included/ignored file filters enable monorepo CI/CD efficiency. │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│           PATTERN 3: EVENT-DRIVEN CODE AUDIT & COMPLIANCE             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Scenario: Regulated industry — every push must be audited and     │
│ scanned for secrets/compliance violations.                         │
│                                                                       │
│ Architecture:                                                       │
│ ┌───────────────────────────────────────────────────────────────┐  │
│ │                                                               │  │
│ │  git push                                                     │  │
│ │       │                                                       │  │
│ │       ▼                                                       │  │
│ │  ┌──────────────┐                                            │  │
│ │  │ CSR repo      │                                            │  │
│ │  └──────┬────────┘                                            │  │
│ │         │                                                     │  │
│ │    ┌────┴────┐                                                │  │
│ │    ▼         ▼                                                │  │
│ │ Pub/Sub   Cloud Build                                        │  │
│ │ topic     (build + test)                                     │  │
│ │    │                                                          │  │
│ │    ▼                                                          │  │
│ │ Cloud Function: "audit-push"                                 │  │
│ │ ├── Log push event to BigQuery (audit trail)                 │  │
│ │ ├── Run truffleHog / gitleaks (secret scanning)              │  │
│ │ ├── Check commit message format (conventional commits)       │  │
│ │ ├── Alert on force push (Slack notification)                 │  │
│ │ └── Tag commit in BigQuery with compliance metadata          │  │
│ │                                                               │  │
│ │    ┌──────────────────────────────────────────────────┐      │  │
│ │    │ BigQuery: "code_audit.push_events"                │      │  │
│ │    │                                                   │      │  │
│ │    │ timestamp | user | repo | branch | commit |      │      │  │
│ │    │ files_changed | secrets_found | compliance_ok    │      │  │
│ │    └──────────────────────────────────────────────────┘      │  │
│ │                                                               │  │
│ │    ┌──────────────────────────────────────────────────┐      │  │
│ │    │ Looker Studio Dashboard                           │      │  │
│ │    │ • Pushes per day / per developer                 │      │  │
│ │    │ • Secret leak incidents                          │      │  │
│ │    │ • Compliance violations over time                │      │  │
│ │    └──────────────────────────────────────────────────┘      │  │
│ │                                                               │  │
│ └───────────────────────────────────────────────────────────────┘  │
│                                                                       │
│ Why CSR for this pattern:                                          │
│ ├── Pub/Sub notifications = event-driven audit pipeline            │
│ ├── All code stays within GCP (data residency / sovereignty)      │
│ ├── IAM controls who can push (org-level governance)               │
│ ├── VPC Service Controls prevent code exfiltration                 │
│ └── Cloud Logging captures all API operations                      │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Quick Reference

```
┌─────────────────────────────────────────────────────────────────────┐
│           CLOUD SOURCE REPOSITORIES QUICK REFERENCE                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What: Fully managed private Git hosting + mirroring in GCP         │
│ Status: ⚠️ DEPRECATED (June 2024) — use GitHub + Cloud Build      │
│                                                                       │
│ Two modes:                                                          │
│ ├── Hosted repo — GCP-native Git repo (read/write)                │
│ └── Mirrored repo — auto-sync from GitHub/Bitbucket (read-only)  │
│                                                                       │
│ Authentication:                                                     │
│ ├── gcloud credential helper (recommended)                         │
│ ├── SSH key (port 2022)                                            │
│ ├── Manually generated credentials                                 │
│ └── Service account (CI/CD)                                        │
│                                                                       │
│ Key integrations:                                                   │
│ ├── Cloud Build — triggers on push/tag                             │
│ ├── Cloud Functions — deploy from repo                             │
│ ├── App Engine — deploy from repo                                  │
│ ├── Pub/Sub — notifications on repo events                        │
│ ├── Error Reporting — source code linking                          │
│ └── Cloud Shell — open repo in browser IDE                        │
│                                                                       │
│ IAM roles:                                                          │
│ ├── source.reader — clone, browse, search                         │
│ ├── source.writer — push, branch, tag                             │
│ └── source.admin — create/delete repos, configure                 │
│                                                                       │
│ Console features:                                                   │
│ ├── Code browser with syntax highlighting                          │
│ ├── Git blame and file history                                     │
│ └── Cross-repo regex search                                        │
│                                                                       │
│ CLI:                                                                │
│ ├── gcloud source repos create <name>                              │
│ ├── gcloud source repos clone <name>                               │
│ ├── gcloud source repos list                                       │
│ └── gcloud source repos delete <name>                              │
│                                                                       │
│ Pricing: 5 free users, then $1/user/month + storage/egress        │
│                                                                       │
│ ⚠️ Mirroring: Console only (no CLI/Terraform for mirror setup).   │
│ ⚠️ No pull requests, code review, issue tracking in CSR.          │
│ ⚠️ IAM is project-level (not per-repo by default).                │
│                                                                       │
│ Modern replacement: GitHub/GitLab + Cloud Build 2nd-gen triggers  │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Migration to GitHub + Cloud Build

Since Cloud Source Repositories is **deprecated (June 2024)**, Google recommends migrating to GitHub (or GitLab) with Cloud Build 2nd-gen triggers. Here's how to do it step by step.

### Step 1: Export Your CSR Repos

CSR repos are standard Git repositories — you can clone them and push to GitHub.

```bash
# List all your CSR repos
gcloud source repos list --project=my-project

# Clone a CSR repo locally (with all branches and tags)
gcloud source repos clone my-app --project=my-project
cd my-app

# Verify all branches came through
git branch -a
```

### Step 2: Create GitHub Repos and Push

```bash
# Create a new repo on GitHub (using GitHub CLI)
gh repo create my-org/my-app --private --confirm

# Add GitHub as a new remote
git remote add github https://github.com/my-org/my-app.git

# Push all branches and tags to GitHub
git push github --all
git push github --tags

# Verify on GitHub
gh repo view my-org/my-app --web
```

**For multiple repos**, script it:

```bash
#!/bin/bash
# migrate-all-repos.sh
PROJECT="my-project"
GITHUB_ORG="my-org"

for REPO in $(gcloud source repos list --project=$PROJECT --format="value(name)"); do
    echo "Migrating $REPO..."
    
    # Clone from CSR
    gcloud source repos clone "$REPO" --project="$PROJECT"
    cd "$REPO"
    
    # Create GitHub repo
    gh repo create "$GITHUB_ORG/$REPO" --private --confirm
    
    # Push everything
    git remote add github "https://github.com/$GITHUB_ORG/$REPO.git"
    git push github --all
    git push github --tags
    
    cd ..
    echo "✅ $REPO migrated"
done
```

### Step 3: Set Up Cloud Build GitHub Triggers

Replace your CSR-based Cloud Build triggers with GitHub-connected triggers.

**Connect GitHub to Cloud Build (one-time setup):**

1. Go to **Cloud Build → Triggers** in the Console
2. Click **Manage repositories** → **Link repository**
3. Select **GitHub (Cloud Build GitHub App)**
4. Authenticate and select your GitHub org/repos
5. Install the Cloud Build GitHub App on your repos

**Create a trigger for your migrated repo:**

```bash
# Create a 2nd-gen trigger connected to GitHub
gcloud builds triggers create github \
    --name="build-my-app" \
    --repo-name="my-app" \
    --repo-owner="my-org" \
    --branch-pattern="^main$" \
    --build-config="cloudbuild.yaml" \
    --region=us-central1
```

**Or from the Console:**

```
Cloud Build → Triggers → CREATE TRIGGER
├── Source: GitHub (2nd gen)
├── Repository: my-org/my-app
├── Event: Push to branch
├── Branch: ^main$
├── Config: cloudbuild.yaml
└── CREATE
```

> **💡 Tip:** Your existing `cloudbuild.yaml` files should work as-is — the build config format is the same regardless of whether the source is CSR or GitHub.

### Step 4: Update CI/CD and Team Workflows

| Old (CSR) | New (GitHub) |
|-----------|--------------|
| `gcloud source repos clone my-app` | `git clone https://github.com/my-org/my-app.git` |
| CSR push triggers | GitHub 2nd-gen triggers |
| IAM `source.writer` role | GitHub collaborator access |
| Console code browser | GitHub web UI (much better) |
| No pull requests | GitHub PRs with code review |
| Pub/Sub notifications | GitHub webhooks + Eventarc |

### Step 5: Clean Up CSR

Once everything is working on GitHub:

```bash
# Verify GitHub triggers are working (do a test push)
git push github main
# Check Cloud Build → History for successful builds

# Delete old CSR triggers
gcloud builds triggers list --filter="triggerTemplate.repoName=my-app"
gcloud builds triggers delete OLD_TRIGGER_ID

# Delete the CSR repo (optional — it's deprecated anyway)
gcloud source repos delete my-app --project=my-project
```

> **⚠️ Warning:** Don't delete CSR repos until you've confirmed all branches, tags, and build triggers are working on GitHub. Keep CSR repos as read-only backups for a transition period.

---

## What's Next?

Continue to **Chapter 31: Cloud Build** → `31-cloud-build.md`
