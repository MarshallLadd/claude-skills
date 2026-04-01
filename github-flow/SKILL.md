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
All work happens in short-lived branches that merge into `development`.
`development` merges into `main` only for releases.

---

## Branch Structure

```
main              ← production / releases only
  └── development ← integration branch; always in a releasable state
        ├── feat/add-user-login       ← your work lives here
        ├── fix/null-on-logout
        └── chore/update-dependencies
```

**Never commit directly to `main` or `development`.** Both branches are
protected — all changes arrive via pull request.

---

## Branch Naming

Format: `type/short-description`

Use kebab-case. Keep descriptions concise but specific enough to understand
at a glance in a branch list.

| Type | When to use |
|------|-------------|
| `feat/` | New feature or user-facing capability |
| `fix/` | Bug fix |
| `chore/` | Maintenance, dependency updates, tooling |
| `docs/` | Documentation changes only |
| `refactor/` | Code restructure with no behavior change |
| `hotfix/` | Urgent production fix (see Hotfixes below) |

**Examples:**
- `feat/add-user-authentication`
- `fix/login-null-pointer`
- `chore/upgrade-typescript-5`
- `docs/update-api-reference`
- `refactor/extract-auth-middleware`

---

## Starting New Work

Always branch from `development` to pick up the latest integrated changes:

```bash
git checkout development
git pull origin development
git checkout -b feat/your-feature-name
```

Push early to create a remote backup and make work visible to collaborators:

```bash
git push -u origin feat/your-feature-name
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

When a branch is ready for review, open a PR targeting `development`:

```bash
gh pr create \
  --base development \
  --title "feat: short description" \
  --body "## What this does

<!-- One paragraph describing the change and why it was made -->

## How to test

<!-- Step-by-step instructions to verify the change works correctly -->

## Notes

<!-- Anything reviewers should pay attention to, edge cases, trade-offs -->"
```

Always target `development`, never `main` — except for releases and hotfixes.

---

## Merging

After review and approval, merge using squash to keep history readable.
Delete the branch after merging — it has served its purpose.

```bash
gh pr merge <pr-number> --squash --delete-branch
```

Use squash for feature and fix branches. Use a regular merge commit for
release PRs (development → main) to preserve the full release history.

---

## Releases — Promoting development to main

A release is a deliberate, intentional promotion of `development` into `main`.
Only do this when `development` is stable and the team agrees it is ready
to ship.

```bash
# Make sure development is up to date locally
git checkout development
git pull origin development

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
- [ ] All tests passing on development
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

For urgent production bugs that cannot wait for the normal development cycle:

```bash
# Branch from main — not development — to avoid pulling in unfinished work
git checkout main
git pull origin main
git checkout -b hotfix/describe-the-critical-issue
```

After fixing and committing, open PRs into **both** `main` and `development`
so the fix is not lost when the next release happens:

```bash
gh pr create --base main \
  --title "hotfix: brief description" \
  --body "## Hotfix

**Problem:** <!-- What broke and what was the impact? -->
**Fix:** <!-- What was changed? -->
**Testing:** <!-- How was this verified? -->"

gh pr create --base development \
  --title "hotfix: (backport) brief description" \
  --body "Backport of hotfix for <issue>. See main PR for details."
```

Merge the `main` PR first, tag the release, then merge the backport into
`development`.

---

## Quick Reference

| Situation | Action |
|-----------|--------|
| Starting any new work | Branch from `development` |
| Work is ready | PR → `development` |
| Shipping a release | PR from `development` → `main`, then tag |
| Urgent production fix | Branch from `main`, PR → `main` + `development` |
| Never | Commit directly to `main` or `development` |
