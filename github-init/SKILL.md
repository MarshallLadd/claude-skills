---
name: github-init
description: >
  One-time Git/GitHub repository initialization for a new project. ALWAYS
  invoke this skill when /init is run, when a new Claude Code project is
  opened that lacks a git repository, or when the user asks to "set up git",
  "initialize the repo", or "push this to GitHub". Covers git init,
  .gitignore creation and security audit, scaffolding commit, optional GitHub
  remote creation and branch setup, and optional branch protections. Do not
  skip this skill if the project has no .git folder.
---

# GitHub Repository Init

This skill runs once per project, during or immediately after `/init`. Its
goal is to get the project into version control with a clean history, secure
defaults, and a proper branch structure before any real code is committed.

---

## Phase 0 — Remote Intent

**Ask the user this first, before doing anything else:**

> "Do you want this repository pushed to GitHub with a remote, or would you
> prefer local-only tracking for now?"

- **Remote (GitHub):** All phases run, including remote creation and branch
  protections.
- **Local only:** Phases 1–4 and Phase 6 (local branches) run. Phases 5 and
  7 are skipped. Branch protections are a GitHub feature and won't apply, but
  the branch structure will still exist locally.

Store the user's choice and refer to it at each phase gate below.

---

## Phase 1 — Git Initialization

Check whether a git repo already exists:

```bash
git rev-parse --is-inside-work-tree 2>/dev/null
```

If the command fails (no repo), initialize one:

```bash
git init
```

---

## Phase 2 — Stack Detection & .gitignore

Detect the project's primary language and stack by inspecting files present:

| File(s) present | Stack |
|-----------------|-------|
| `package.json` | Node / JavaScript / TypeScript |
| `requirements.txt`, `pyproject.toml`, `Pipfile` | Python |
| `*.csproj`, `*.sln` | .NET / C# |
| `Gemfile` | Ruby |
| `go.mod` | Go |
| `pom.xml`, `build.gradle` | Java / Kotlin |
| `Cargo.toml` | Rust |

Use the detected stack to select the appropriate GitHub-maintained .gitignore
template as a base. For multi-language projects, layer relevant templates.

**Then append these security-focused enhancements** after the template:

```gitignore
# ── Security & Secrets ──────────────────────────────────────────────
# Environment and credential files — never commit live secrets
.env
.env.*
!.env.example
*.pem
*.key
*.p12
*.pfx
secrets/
credentials/
config/secrets.*

# API key and token files that commonly slip through
*.token
auth.json
serviceAccountKey.json

# Editor and OS artifacts
.DS_Store
Thumbs.db
*.swp
*~
.vscode/settings.json
!.vscode/extensions.json
.idea/

# Local Claude Code overrides (personal settings, not project config)
.claude/settings.local.json
```

**If a `.gitignore` already exists:** audit it against the list above and
suggest any gaps. Explain each suggestion briefly. Apply changes only after
the user confirms.

Add a comment block at the top of the .gitignore file:

```gitignore
# .gitignore — <ProjectName>
# Base: <detected stack> template (github.com/github/gitignore)
# Security enhancements added by github:init
# Review this file before committing any new file type to the project.
```

---

## Phase 3 — Scaffolding Files

Ensure these files exist. Create them if missing — do not overwrite files
that already have meaningful content.

**README.md** — scaffold only, user fills in details later:

```markdown
# <ProjectName>

<!-- One-line description of this project -->

## Overview

<!-- What does this project do? Why does it exist? -->

## Getting Started

<!-- Prerequisites, installation steps, and first-run instructions -->

## Development

<!-- How to run locally, run tests, build, etc. -->
<!-- See CLAUDE.md for branch and PR conventions. -->

## License

<!-- License type, e.g. MIT, proprietary -->
```

**CLAUDE.md** — should already exist from `/init`. If it does not, note the
gap but do not block progress; the user can regenerate it with `/init`.

---

## Phase 4 — Initial Commit

Stage only the scaffolding files — no source code, no compiled output:

```bash
git add .gitignore README.md CLAUDE.md
git commit -m "chore: initial project scaffolding

- .gitignore: <stack> base + security enhancements
- README.md: project scaffold
- CLAUDE.md: Claude Code configuration"
```

The first commit should contain no application code. This creates a clean,
reviewable starting point and ensures secrets can never appear in the initial
history.

---

## Phase 5 — GitHub Remote  *(skip if local-only)*

### Determine the repository name

Derive a candidate name by checking these sources in order, stopping at the
first meaningful result:

1. `package.json` → `name` field
2. `pyproject.toml` → `[project] name` or `[tool.poetry] name`
3. `*.csproj` → `<RootNamespace>` or `<AssemblyName>` element
4. `CLAUDE.md` → project name stated at the top of the file
5. Working directory folder name (strip common suffixes like `-app`, `-project`)

Clean the candidate: lowercase, replace spaces and underscores with hyphens,
remove special characters, max 100 characters.

**Always present the derived name and ask for confirmation:**

> "I suggest naming the GitHub repository `<derived-name>` (from
> `<source>`). Press Enter to accept, or type a different name:"

Use whatever the user confirms.

### Create the remote

Ask:
1. Personal account or an organization? (if org, which one?)
2. Public or private?

```bash
gh repo create <confirmed-name> --<public|private> \
  --source=. --remote=origin --push
```

---

## Phase 6 — Branch Structure

### Remote repositories

After the initial push, create and push the `development` branch:

```bash
git checkout -b development
git push -u origin development
```

Set `development` as the default branch on GitHub (where PRs target):

```bash
gh repo edit --default-branch development
```

### Local-only repositories

Create both protected branches locally so the structure exists even without
a remote:

```bash
git checkout -b development
git checkout main
```

Note to user: branch protections are enforced on GitHub. For local-only
repos, the convention still applies — avoid committing directly to `main` or
`development`.

---

## Phase 7 — Branch Protections  *(skip if local-only)*

Apply protections to both branches. First, resolve the repo owner and name:

```bash
REPO=$(gh repo view --json nameWithOwner -q .nameWithOwner)
OWNER=$(echo $REPO | cut -d/ -f1)
NAME=$(echo $REPO | cut -d/ -f2)
```

**`main`** — production branch, releases only. Direct pushes blocked.

```bash
gh api repos/$OWNER/$NAME/branches/main/protection \
  --method PUT \
  --field enforce_admins=true \
  --field required_status_checks=null \
  --field restrictions=null \
  --field 'required_pull_request_reviews[required_approving_review_count]=0'
```

**`development`** — integration branch. PRs required, direct pushes blocked.

```bash
gh api repos/$OWNER/$NAME/branches/development/protection \
  --method PUT \
  --field enforce_admins=false \
  --field required_status_checks=null \
  --field restrictions=null \
  --field 'required_pull_request_reviews[required_approving_review_count]=0'
```

Confirm protections applied:

```bash
gh api repos/$OWNER/$NAME/branches/main/protection \
  --jq '.required_pull_request_reviews'
gh api repos/$OWNER/$NAME/branches/development/protection \
  --jq '.required_pull_request_reviews'
```

---

## Phase 8 — Catch-up Commit  *(if project already has code)*

If the working directory contains files beyond the scaffolding (source code,
config, assets, etc.), collect them into a single catch-up commit and open
a PR targeting `development`:

```bash
git checkout -b feat/initial-codebase
git add .
git commit -m "feat: add initial project codebase

Bulk commit of pre-existing project files.
All subsequent changes will follow the branch → PR → merge workflow."
git push -u origin feat/initial-codebase
gh pr create \
  --base development \
  --title "feat: initial codebase" \
  --body "Bulk import of existing project files.

All future changes will use the standard branch → PR → merge workflow.
See CLAUDE.md for branching conventions."
```

For local-only repos, commit to `feat/initial-codebase` and note the PR step
is skipped — the user can open one when they add a remote later.

---

## Completion Checklist

Run through this before declaring init complete:

- [ ] `.git` directory exists
- [ ] `.gitignore` present and security-reviewed
- [ ] `README.md` exists with project scaffold
- [ ] `CLAUDE.md` exists
- [ ] Initial commit contains only scaffolding (no source code)
- [ ] `main` and `development` branches exist
- [ ] *(remote)* Remote `origin` is set and reachable
- [ ] *(remote)* `development` is the default branch on GitHub
- [ ] *(remote)* Branch protections active on `main` and `development`
- [ ] *(if code existed)* Catch-up commit made; PR opened to `development`

Report the GitHub repo URL (if remote) or the local `.git` path (if
local-only) to the user when done.
