# Chapter 34: Azure DevOps - Artifacts

---

## Table of Contents

- [Overview](#overview)
- [Part 1: Artifacts Fundamentals](#part-1-artifacts-fundamentals)
- [Part 2: Creating a Feed (Portal Walkthrough)](#part-2-creating-a-feed-portal-walkthrough)
- [Part 3: Package Types](#part-3-package-types)
- [Part 4: Upstream Sources](#part-4-upstream-sources)
- [Part 5: Publishing from Pipelines](#part-5-publishing-from-pipelines)
- [Part 6: Consuming Packages](#part-6-consuming-packages)
- [Part 7: az CLI Reference](#part-7-az-cli-reference)
- [Quick Reference](#quick-reference)
- [What's Next?](#whats-next)

---

## Overview

Azure Artifacts lets you create and share packages (npm, NuGet, Maven, Python, Universal) within your organization. Think of it as a private package registry — like a private npm or NuGet feed that only your team can access. You can also set up upstream sources to cache packages from public registries.

```
What you'll learn:
├── Artifacts Fundamentals
│   ├── What are package feeds
│   ├── Why use a private registry
│   └── Feed scopes (project vs organization)
├── Creating a Feed (Portal)
├── Package Types (npm, NuGet, Maven, Python, Universal)
├── Upstream Sources (proxy public registries)
├── Publishing from Pipelines (CI integration)
├── Consuming Packages (configure client tools)
└── az CLI reference
```

---

## Part 1: Artifacts Fundamentals

```
┌─────────────────────────────────────────────────────────────────────┐
│           AZURE ARTIFACTS CONCEPTS                                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What is a Feed?                                                      │
│ A feed is a private package store (like a private npm registry).   │
│                                                                       │
│ Why use private feeds?                                               │
│ ├── Share internal packages across teams                          │
│ ├── Control which versions your team uses                        │
│ ├── Cache public packages (faster, more reliable builds)         │
│ ├── Security: scan packages before use                            │
│ └── Compliance: audit trail of package usage                     │
│                                                                       │
│ Feed scopes:                                                         │
│ ├── Project-scoped: Visible only within one project              │
│ └── Organization-scoped: Visible across all projects             │
│                                                                       │
│ Supported package types:                                             │
│ ├── npm (JavaScript/TypeScript)                                   │
│ ├── NuGet (.NET/C#)                                               │
│ ├── Maven (Java)                                                   │
│ ├── Gradle (Java)                                                  │
│ ├── Python (pip)                                                   │
│ └── Universal Packages (any file type)                            │
│                                                                       │
│ Pricing:                                                             │
│ ├── Free: 2 GB per organization                                  │
│ └── Extra storage: Paid per GB                                   │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 2: Creating a Feed (Portal Walkthrough)

```
Project → Artifacts → Create Feed

┌─────────────────────────────────────────────────────────────────────┐
│           CREATE FEED                                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Name: [mycompany-packages]                                          │
│                                                                       │
│ Visibility:                                                          │
│ ● Members of [myproject]                                           │
│ ○ Members of [mycompany] (organization)                           │
│                                                                       │
│ Scope:                                                               │
│ ● Project-scoped (recommended for team packages)                  │
│ ○ Organization-scoped (shared across projects)                   │
│                                                                       │
│ Upstream sources:                                                    │
│ ☑ Include packages from common public sources                    │
│   ⚡ Proxies npmjs.org, nuget.org, pypi.org, Maven Central      │
│   ⚡ Packages from public sources are cached in your feed!      │
│                                                                       │
│ [Create]                                                             │
│                                                                       │
│ After creation:                                                      │
│ Feed URL: https://pkgs.dev.azure.com/mycompany/myproject/         │
│           _packaging/mycompany-packages/npm/registry/              │
│                                                                       │
│ Click [Connect to feed] for setup instructions per package type  │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 3: Package Types

### npm

```bash
# Configure .npmrc in your project root
@mycompany:registry=https://pkgs.dev.azure.com/mycompany/myproject/_packaging/mycompany-packages/npm/registry/
always-auth=true

# Authenticate (one-time)
npx vsts-npm-auth -config .npmrc

# Publish package
npm publish

# Install from feed
npm install @mycompany/shared-utils
```

### NuGet (.NET)

```bash
# Add source
dotnet nuget add source https://pkgs.dev.azure.com/mycompany/myproject/_packaging/mycompany-packages/nuget/v3/index.json \
  --name MyFeed \
  --username anything \
  --password <PAT>

# Publish
dotnet nuget push MyPackage.1.0.0.nupkg --source MyFeed

# Install
dotnet add package MyCompany.SharedUtils
```

### Python (pip)

```bash
# Install from feed
pip install my-package \
  --index-url https://pkgs.dev.azure.com/mycompany/myproject/_packaging/mycompany-packages/pypi/simple/

# Publish with twine
twine upload --repository-url https://pkgs.dev.azure.com/mycompany/myproject/_packaging/mycompany-packages/pypi/upload/ \
  dist/*
```

### Universal Packages (any files)

```bash
# Publish any folder as a package
az artifacts universal publish \
  --organization https://dev.azure.com/mycompany \
  --feed mycompany-packages \
  --name my-artifact \
  --version 1.0.0 \
  --path ./build-output/

# Download
az artifacts universal download \
  --organization https://dev.azure.com/mycompany \
  --feed mycompany-packages \
  --name my-artifact \
  --version 1.0.0 \
  --path ./download/
```

---

## Part 4: Upstream Sources

```
┌─────────────────────────────────────────────────────────────────────┐
│           UPSTREAM SOURCES                                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Feed Settings → Upstream sources                                   │
│                                                                       │
│ How it works:                                                        │
│                                                                       │
│ Developer → npm install lodash                                    │
│     │                                                                │
│     ▼                                                                │
│ Your Feed (checks here first)                                      │
│     │ Not found locally                                             │
│     ▼                                                                │
│ Upstream: npmjs.org (fetches + caches)                             │
│     │                                                                │
│     ▼                                                                │
│ Package saved in your feed (cached!)                               │
│                                                                       │
│ Benefits:                                                            │
│ ├── Faster: Cached locally after first download                  │
│ ├── Reliable: Builds work even if npm is down                    │
│ ├── Secure: Can block specific vulnerable packages               │
│ └── Single source: All packages (private + public) from one URL │
│                                                                       │
│ Default upstream sources:                                            │
│ ├── npmjs.org → npm packages                                     │
│ ├── nuget.org → NuGet packages                                   │
│ ├── pypi.org → Python packages                                   │
│ └── Maven Central → Java packages                                │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 5: Publishing from Pipelines

```yaml
# azure-pipelines.yml — publish npm package
trigger:
  - main

pool:
  vmImage: 'ubuntu-latest'

steps:
  - task: NodeTool@0
    inputs:
      versionSpec: '20.x'

  - script: npm ci
    displayName: 'Install dependencies'

  - script: npm test
    displayName: 'Run tests'

  - script: npm run build
    displayName: 'Build package'

  - task: Npm@1
    displayName: 'Publish to Artifacts feed'
    inputs:
      command: 'publish'
      publishRegistry: 'useFeed'
      publishFeed: 'myproject/mycompany-packages'
```

```yaml
# Publish NuGet from pipeline
steps:
  - task: DotNetCoreCLI@2
    inputs:
      command: 'pack'
      packagesToPack: '**/*.csproj'
      versioningScheme: 'byBuildNumber'

  - task: NuGetCommand@2
    inputs:
      command: 'push'
      publishVstsFeed: 'myproject/mycompany-packages'
```

---

## Part 6: Consuming Packages

```
Feed → Connect to feed → Choose package type

┌─────────────────────────────────────────────────────────────────────┐
│           CONNECT TO FEED                                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ npm:                                                                 │
│   1. Add .npmrc file (generated instructions shown)               │
│   2. Run: npx vsts-npm-auth -config .npmrc                       │
│   3. npm install @mycompany/package-name                          │
│                                                                       │
│ NuGet:                                                               │
│   1. Add nuget.config file (generated)                             │
│   2. dotnet add package MyCompany.PackageName                     │
│                                                                       │
│ pip:                                                                 │
│   1. pip install package-name --index-url <feed-url>              │
│   2. Or add to pip.conf / requirements.txt                        │
│                                                                       │
│ In pipelines (no auth needed!):                                     │
│   Pipelines automatically authenticate to same-org feeds.        │
│                                                                       │
│ Views (package promotion):                                          │
│ ├── @Local → All packages (dev/CI builds)                       │
│ ├── @Prerelease → Promote here for testing                      │
│ └── @Release → Promote here for stable releases                 │
│ ⚡ Views let you separate dev builds from stable releases.      │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 7: az CLI Reference

```bash
# Install DevOps extension
az extension add --name azure-devops

# List feeds
az artifacts feed list --output table

# Create feed
az artifacts feed create --name mycompany-packages

# Publish universal package
az artifacts universal publish \
  --feed mycompany-packages \
  --name my-artifact \
  --version 1.0.0 \
  --path ./output/

# Download universal package
az artifacts universal download \
  --feed mycompany-packages \
  --name my-artifact \
  --version 1.0.0 \
  --path ./download/

# Delete feed
az artifacts feed delete --name mycompany-packages --yes
```

---

## Real-World Patterns

### Pattern 1: Private Package Feed for Organization

```
┌─────────────────────────────────────────────────┐
│  Internal Package Management                    │
├─────────────────────────────────────────────────┤
│                                                 │
│  Shared Library Team                            │
│       │                                         │
│       ▼                                         │
│  Build Pipeline ─→ Artifacts Feed               │
│  (publishes NuGet/npm)   │                      │
│                    ┌────┴────┐                  │
│                    │ Upstream │                  │
│                    │ Sources  │                  │
│                    ├─────────┤                  │
│                    │ nuget.org│                  │
│                    │ npmjs.com│                  │
│                    └────┬────┘                  │
│         ┌──────────┼──────────┐                 │
│         ▼          ▼          ▼                 │
│     Team A     Team B     Team C                │
│     (Web)      (API)      (Mobile)              │
│                                                 │
│  One feed for internal + public packages.       │
│  Retention: keep latest 3 versions.             │
└─────────────────────────────────────────────────┘
```

---

## Quick Reference

```
Feed = Private package registry (like private npm/NuGet/PyPI)
Upstream = Proxy that caches packages from public registries

Package types: npm, NuGet, Maven, Python (pip), Universal
Feed scope: Project-scoped or Organization-scoped
Free tier: 2 GB storage per organization

Views (promotion): @Local → @Prerelease → @Release
Pipeline auth: Automatic within same organization
Dev auth: PAT (Personal Access Token) or vsts-npm-auth

Connect: Feed → Connect to feed → Follow instructions
```

---

## What's Next?

Next chapter: [Chapter 35: Azure Container Registry (ACR)](35-acr.md) — Private Docker registry with SKUs, geo-replication, tasks, and vulnerability scanning.
