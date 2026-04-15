# Engineer Workflow — Staging → Prod Sync

## Mental model
- **`staging-app`** is where you work. Treat it as normal.
- **`prod-app`** is downstream. You never push to it directly.
- Every push to staging `main` → auto-PR on prod `main`. Every merge on prod → auto-mirror back to staging.

---

## Daily workflow

1. **Clone staging only:**
   ```bash
   git clone https://github.com/vkpriyesh/staging-app.git
   ```
   You don't need a local clone of prod-app.

2. **Work on a feature branch off staging `main`:**
   ```bash
   git checkout -b feat/my-change
   # edit, commit
   git push -u origin feat/my-change
   ```

3. **Open a PR into staging `main`.** Normal review rules apply — linting, tests, approvals. This is the *real* review gate.

4. **Merge the PR on staging.** The `sync-to-prod` workflow fires automatically:
   - Clones prod into a temp dir
   - Creates branch `sync/staging-<timestamp>`
   - Mirrors staging's files (minus `.git`, `.github`, `node_modules`) into prod
   - Opens a PR titled `Staging sync YYYY-MM-DD` on prod-app
   - **If nothing changed vs prod main, the job exits cleanly with no PR.**

5. **Review the prod PR.** Usually this is a rubber-stamp — it's the same diff that already passed review on staging. If prod has policies requiring extra approval (compliance, release manager sign-off), handle that here.

6. **Merge the prod PR (use "Squash and merge").** The `sync-back-to-staging` workflow fires:
   - Mirrors prod's current state back to staging `main`
   - Commits as `sync-back: from prod YYYY-MM-DD` and pushes
   - If nothing changed, exits cleanly with "no changes"

7. **Pull the sync-back on your local staging** before starting the next feature:
   ```bash
   git checkout main && git pull
   ```

---

## Rules

| Rule | Why |
|---|---|
| Never push directly to `prod-app` main | Breaks the audit trail; your commit won't sync back and will be overwritten by the next staging sync |
| Never push directly to a `sync/staging-*` branch | Those are workflow-managed; add follow-up commits on a normal feature branch instead |
| Never start a staging commit message with `sync-back:` | It'll be mistaken for a sync-back loop commit and skipped |
| Never start a prod commit message with `sync:` | Same — the sync-back guard will skip it |
| Edits to `.github/workflows/*` don't cross-sync | rsync excludes `.github/`. Workflow files are per-repo; update each repo directly |
| Don't commit `prod-repo/` or `staging-repo/` folders | They're reserved names used by the workflows |

---

## When things go wrong

**"Staging sync" PR on prod has a merge conflict.**
Conflicts on prod mean someone edited prod directly (bad) or a previous sync-back commit contains edits not yet pulled to your staging clone. Fix:
```bash
cd staging-app
git pull                              # get latest sync-back
# resolve in a new feature branch, merge to staging main
```
Next staging push overwrites the conflict cleanly.

**Sync-back created a commit I don't recognize on staging.**
It came from a prod-side edit during PR review. Check the prod PR history to see who edited what. The commit message will be `sync-back: from prod YYYY-MM-DD`.

**Workflow failed on GitHub Actions.**
Check the run log. Most common causes:
- `GH_PAT` expired → regenerate PAT, update secret on both repos
- Orphan branch left behind on prod → delete via GitHub UI or `git push origin --delete sync/staging-<timestamp>`

**I need to bypass the sync (hotfix directly on prod).**
Strongly discouraged. If absolutely needed:
1. Make the fix on prod via a normal PR
2. Merge it — sync-back will mirror to staging
3. Verify staging now matches

**I need to test a workflow change.**
Workflow files are per-repo and not synced. Edit `.github/workflows/*.yml` on the relevant repo's own branch, push, open a PR in that repo, merge. Use `workflow_dispatch` to trigger manually for testing.

---

## What lives where

| | `staging-app` | `prod-app` |
|---|---|---|
| Source of truth for content | Yes | No (mirrored) |
| Engineers push here | Yes | No |
| Feature branches | Yes | No (only `sync/*` branches, auto-managed) |
| Workflow file | `sync-to-prod.yml` | `sync-back-to-staging.yml` |
| Secrets needed | `GH_PAT` | `GH_PAT` |
