---
name: github-flow
description: >
  Ongoing Git branching, commit, and PR conventions for projects initialized
  with github:init. Invoke this skill whenever the user wants to start new
  work ("I'm going to add X", "let's fix Y"), create a branch, write a commit
  message, open a PR, or cut a release. Also invoke when the user asks how
  branching works, how to merge, or how to do a release. Acts as the standing
  rulebook for day-to-day Git workflow on this project.
---

# GitHub Flow — Branch & PR Conventions

This skill defines how work moves through the repository after initialization.
All work happens in short-lived branches that merge into `develop`.
`develop` merges into `main` only for releases.

---

## Branch Structure

```
main              ← production / releases only
  └── develop ← integration branch; always in a releasable state
        ├── feature/add-user-login    ← your work lives here
        ├── bugfix/null-on-logout
        └── chore/update-dependencies
```

**Never commit directly to `main` or `develop`.** Both branches are
protected — all changes arrive via pull request.

---

## Branch Naming

Format: `type/short-description`

Use kebab-case. Keep descriptions concise but specific enough to understand
at a glance in a branch list.

| Type | When to use |
|------|-------------|
| `feature/` | New feature or user-facing capability |
| `bugfix/` | Bug fix |
| `chore/` | Maintenance, dependency updates, tooling |
| `docs/` | Documentation changes only |
| `refactor/` | Code restructure with no behavior change |
| `hotfix/` | Urgent production fix (see Hotfixes below) |

**Examples:**
- `feature/add-user-authentication`
- `bugfix/login-null-pointer`
- `chore/upgrade-kotlin-2`
- `docs/update-api-reference`
- `refactor/extract-auth-middleware`

---

## Starting New Work

Always branch from `develop` to pick up the latest integrated changes:

```bash
git checkout develop
git pull origin develop
git checkout -b feature/your-feature-name
```

Push early to create a remote backup and make work visible to collaborators:

```bash
git push -u origin feature/your-feature-name
```

---

## Commit Messages

Follow Conventional Commits format: `type: brief description`

Write the description in imperative mood (as if completing "This commit
will..."), lowercase, no trailing period.

```
feat: add JWT authentication middleware
fix: resolve null reference in user login flow
chore: update ESLint to v9
docs: add environment setup instructions to README
refactor: extract token validation into its own module
```

For commits where the why matters, add a body after a blank line:

```
fix: prevent duplicate session tokens on refresh

Token refresh was issuing a new token without invalidating the previous
one, allowing both tokens to be valid simultaneously. Now explicitly
revokes the old token before issuing a replacement.
```

Keep the subject line under 72 characters. The body can be as long as needed.

---

## Opening a Pull Request

When a branch is ready for review, open a PR targeting `develop`:

```bash
gh pr create \
  --base develop \
  --title "feat: short description" \
  --body "## What this does

<!-- One paragraph describing the change and why it was made -->

## How to test

<!-- Step-by-step instructions to verify the change works correctly -->

## Notes

<!-- Anything reviewers should pay attention to, edge cases, trade-offs -->"
```

Always target `develop`, never `main` — except for releases and hotfixes.

---

## Merging

After review and approval, merge using squash to keep history readable.
Delete the branch after merging — it has served its purpose.

```bash
gh pr merge <pr-number> --squash --delete-branch
```

Use squash for feature and fix branches. Use a regular merge commit for
release PRs (develop → main) to preserve the full release history.

---

## Releases — Promoting develop to main

A release is a deliberate, intentional promotion of `develop` into `main`.
Only do this when `develop` is stable and the team agrees it is ready
to ship.

```bash
# Make sure develop is up to date locally
git checkout develop
git pull origin develop

# Open the release PR
gh pr create \
  --base main \
  --title "release: v<version>" \
  --body "## Release v<version>

### Summary
<!-- What is in this release? -->

### Changes
<!-- Bullet list of significant changes since last release -->

### Checklist
- [ ] All tests passing on develop
- [ ] No open critical bugs
- [ ] Version number bumped
- [ ] CHANGELOG updated (if maintained)"
```

After the PR merges, tag the release on `main`:

```bash
git checkout main
git pull origin main
git tag -a v<version> -m "Release v<version>"
git push origin v<version>
```

---

## Hotfixes

For urgent production bugs that cannot wait for the normal develop cycle:

```bash
# Branch from main — not develop — to avoid pulling in unfinished work
git checkout main
git pull origin main
git checkout -b hotfix/describe-the-critical-issue
```

After fixing and committing, open PRs into **both** `main` and `develop`
so the fix is not lost when the next release happens:

```bash
gh pr create --base main \
  --title "hotfix: brief description" \
  --body "## Hotfix

**Problem:** <!-- What broke and what was the impact? -->
**Fix:** <!-- What was changed? -->
**Testing:** <!-- How was this verified? -->"

gh pr create --base develop \
  --title "hotfix: (backport) brief description" \
  --body "Backport of hotfix for <issue>. See main PR for details."
```

Merge the `main` PR first, tag the release, then merge the backport into
`develop`.

---

## Quick Reference

| Situation | Action |
|-----------|--------|
| Starting any new work | Branch from `develop` using `feature/`, `bugfix/`, etc. |
| Work is ready | PR → `develop` |
| Shipping a release | PR from `develop` → `main`, then tag |
| Urgent production fix | Branch from `main`, PR → `main` + `develop` |
| Never | Commit directly to `main` or `develop` |
