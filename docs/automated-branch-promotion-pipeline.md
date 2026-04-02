# Lessons Learned: Automated Branch Promotion with RELEASE_NOTES Conflict Resolution

## Context

Three-branch Git flow: `feature/* → development → staging → main`. Automated CI/CD promotes code between branches, bumps versions on main, and syncs version files back. Self-hosted CI (Woodpecker CI on GCP spot VMs) with GitHub branch protection (`enforce_admins: true`, required status checks, 0 reviews).

## The Problem

Automated promotion between branches consistently produced merge conflicts in `RELEASE_NOTES.md`. Every staging→main promotion required manual intervention.

**Root cause chain:**
1. `version-bump.sh` on main moves items from `## Unreleased` to `## vX.Y.Z`
2. `sync-back.sh` syncs updated RELEASE_NOTES back to dev/staging
3. New items accumulate under `## Unreleased` on dev/staging
4. Next staging→main promotion conflicts: main has empty Unreleased, staging has items
5. GitHub's merge API does **not** respect `.gitattributes` merge strategies

**Cascading failures:**
- Sync-back merge to development triggers the promote pipeline
- Promote creates a new staging→main PR before sync-back finishes
- This triggers another version bump, creating empty releases
- Repeat until manual intervention breaks the cycle

## Solution: Always-Local Merge Branch

**Core insight:** `.gitattributes` `merge=union` works for local `git merge` but NOT for GitHub's merge API. The fix is to never create direct branch-to-branch PRs on GitHub. Instead, always merge locally on the CI agent, push a merge branch, and create the PR from that.

### promote.sh Architecture

```
1. Guards (commit message, in-flight PRs, existing PRs, no new commits)
2. git fetch --unshallow  (full history for merge base)
3. git checkout -b merge/{source}-to-{target} origin/{target}
4. git merge origin/{source}  (merge=union handles RELEASE_NOTES)
5. Auto-resolve RELEASE_NOTES.md if conflict (keep source version)
6. Tree-identity check (skip if no actual file changes)
7. Dedup ## Unreleased headers (safety net)
8. git push origin merge/{source}-to-{target}
9. Create PR from merge branch to target
10. Auto-merge (dev→staging) or request reviewer (staging→main)
```

### Three Cascade Guards

```bash
# Guard 1: Skip sync-back and release commits (zero API calls)
if echo "$COMMIT_MSG" | grep -qiE '^Sync:'; then exit 0; fi
if echo "$COMMIT_MSG" | grep -qE '^release: v[0-9]'; then exit 0; fi

# Guard 2: Skip if sync/release PRs are open (single API call)
INFLIGHT=$(curl -s ... | jq '[.[] | select(
    (.title | test("^Sync:")) or (.head.ref | test("^release/"))
  )] | length')
if [ "$INFLIGHT" -gt 0 ]; then exit 0; fi

# Guard 3: Skip if merge produces no file changes (after merge)
DIFF_FILES=$(git diff --name-only "origin/${BASE}" HEAD)
if [ -z "$DIFF_FILES" ]; then exit 0; fi
```

## Supporting Changes

### `.gitattributes`
```
RELEASE_NOTES.md merge=union
```
Tells local `git merge` to keep lines from both sides. This is what makes the local merge branch approach work — GitHub ignores it, but our CI agent respects it.

### Deep Clone in CI
```yaml
# promote.yaml
clone:
  - name: clone
    settings:
      depth: 0
```
Plus `git fetch --unshallow` in the script. Shallow clones cannot find merge bases when branches have diverged commit histories.

### sync-back.sh Dedup Fix
Check if `## Unreleased` exists before inserting. Dedup ALL `##` headers, not just version headers:
```bash
awk '!seen[$0]++ || !/^## /' RELEASE_NOTES.md > tmp && mv tmp RELEASE_NOTES.md
```

### Remove Pre-push Hook
Pre-push hooks that run test suites cause SSH connection timeouts during `git push` (GitHub closes the connection before tests finish). Move all quality checks to the pre-commit hook.

## Key Learnings

1. **GitHub's merge API ignores `.gitattributes`** — the single most important fact. Any merge strategy that relies on `.gitattributes` must happen locally, not through GitHub's UI or API.

2. **Shallow clones break `git merge`** — CI agents typically clone with `depth: 1`. When branches have diverged histories (common after version-bump + sync-back), `git merge` can't find the merge base and produces empty or incorrect results. Always use full clones for merge operations.

3. **Cascading pipelines need circuit breakers** — when pipeline A triggers pipeline B which triggers pipeline A, you need guards at every entry point. Commit-message guards are the cheapest (zero API calls). In-flight PR guards catch timing windows. Both are needed.

4. **Bootstrap problem is unavoidable** — when the fix is in a script that runs on the target branch, the first promotion after deploying the fix will use the OLD script (on the target branch). One manual merge is needed to get the new script onto all branches. After that, it's fully automated.

5. **Tree SHA comparison catches empty promotions** — even when commit counts show branches are "ahead", the actual file content may be identical (common after sync-back). Comparing tree SHAs or using `git diff --name-only` prevents creating empty PRs.

6. **`merge=union` creates duplicate headers** — when both sides have `## Unreleased`, union keeps both lines. A dedup pass after merge is needed as a safety net.

7. **Pre-push hooks and SSH don't mix** — if a pre-push hook runs tests (10+ seconds), the SSH connection to the remote may timeout. Move quality checks to pre-commit and delete the pre-push hook.

## Files Reference

| File | Purpose |
|------|---------|
| `scripts/woodpecker/promote.sh` | Always-local merge branch promotion with three cascade guards |
| `scripts/woodpecker/sync-back.sh` | Version sync with dedup-safe `## Unreleased` insertion |
| `.woodpecker/promote.yaml` | Deep clone (`depth: 0`) for merge base detection |
| `.gitattributes` | `merge=union` for RELEASE_NOTES.md |
| `.githooks/pre-commit` | All quality checks (format, lint, test) |

## Metrics

Before: Every staging→main promotion required manual merge branch creation (5-10 minutes per promotion, multiple times per release).

After: Full automated cycle from feature merge to production tag with zero manual intervention. Three branches align automatically after each release.
