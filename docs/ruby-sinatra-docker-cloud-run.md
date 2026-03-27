# Ruby + Sinatra + Docker on GCP Cloud Run

A practical guide based on deploying the Peregrine Penetrator Reporter — a Sinatra 4 API service that generates penetration test reports (HTML + PDF), running on Cloud Run with Docker images in Artifact Registry and CI/CD via Woodpecker CI.

## The Stack

| Component | Choice | Why |
|-----------|--------|-----|
| Framework | Sinatra 4.2 + Sequel ORM | Lightweight — no Rails overhead for an API service |
| PDF Engine | Chromium headless (primary) + pandoc/xelatex (fallback) | SVG chart support, exact HTML fidelity |
| Container | Docker multi-stage | Baked base image + thin app layer |
| Registry | GCP Artifact Registry | Private Docker images |
| Runtime | Cloud Run | Scale-to-zero, managed TLS, health probes |
| CI/CD | Woodpecker CI (self-hosted) | 98% cheaper than GitHub Actions |

## Baked Base Image Pattern

### Problem

Every Docker build installed texlive (~400MB), chromium (~300MB), ruby gems with native extensions, and system libraries. Builds took 15+ minutes, frequently killed by spot VM reclamation.

### Solution

Split into two Dockerfiles:

**`Dockerfile.base`** — rebuild only when Gemfile.lock or system deps change:
```dockerfile
FROM ruby:3.2.2-slim
RUN apt-get update -qq && apt-get install -y \
    pandoc texlive-xetex texlive-fonts-recommended texlive-fonts-extra \
    chromium fonts-dejavu libsqlite3-0 libpq5 curl \
    build-essential git pkg-config libsqlite3-dev libpq-dev && ...
COPY Gemfile Gemfile.lock ./
RUN bundle install --without development test
RUN apt-get remove -y build-essential ... && apt-get autoremove -y
```

**`Dockerfile`** — thin app layer:
```dockerfile
ARG REPORTER_BASE_IMAGE=reporter-base:latest
FROM ${REPORTER_BASE_IMAGE}
COPY . .
RUN bundle install --without development test 2>/dev/null || true
```

**`build.sh`** — auto-bakes base if missing:
```bash
if gcloud artifacts docker images describe "${BASE_IMAGE}" --quiet 2>/dev/null; then
  echo "Using baked base image"
else
  echo "Base not found — building..."
  docker buildx build -f docker/Dockerfile.base -t "${BASE_IMAGE}" --push .
fi
docker buildx build -f docker/Dockerfile --build-arg "REPORTER_BASE_IMAGE=${BASE_IMAGE}" ...
```

### Results

| Metric | Before | After |
|--------|--------|-------|
| Build time | 446s (7.5 min) | 10s |
| First build (bakes base) | 446s | 446s (one-time) |
| Subsequent builds | 446s | 10s |
| Speedup | — | **45x** |

## Gotcha #1: Sinatra 4 Host Authorization Blocks Cloud Run URLs

**The most time-consuming issue.** Sinatra 4 enables `Rack::Protection::HostAuthorization` by default. Cloud Run generates URLs like `*.a.run.app` which aren't in the permitted hosts list. Every request returns 403 "Host not permitted" — which looks identical to a Cloud Run IAM rejection.

**Symptoms:** 403 on every endpoint including `/health`. Easy to misdiagnose as a Cloud Run IAM issue.

**Fix:**
```ruby
class MyApp < Sinatra::Base
  set :host_authorization, allow_if: ->(_env) { true }
end
```

**Note:** `permitted_hosts` only accepts `String` or `IPAddr` — not symbols, regexes, or `:any`. The `allow_if` proc is the only way to disable it entirely.

## Gotcha #2: Cloud Run IAM vs App-Level Auth — Pick One

Cloud Run can enforce IAM authentication (`roles/run.invoker`) at the infrastructure layer. If your app also has its own Bearer token auth (e.g., `AuthMiddleware` checking `SCAN_CALLBACK_SECRET`), you get a double-auth problem:

- Cloud Run validates the `Authorization` header as an identity token
- Your app validates the same `Authorization` header as an API key
- You can't send both in one header

**Solution:** Set `allUsers → roles/run.invoker` on Cloud Run services and let the app handle auth. Cloud Run provides TLS and scaling; the app handles authorization.

```bash
gcloud run services add-iam-policy-binding SERVICE_NAME \
  --member="allUsers" --role="roles/run.invoker" \
  --region=us-central1 --project=PROJECT
```

## Gotcha #3: Cross-Project Registry Mismatch

If your Docker images are in one GCP project (e.g., `my-project-dev`) and Cloud Run services in another (e.g., `my-project`), scripts that construct registry paths from the Cloud Run project will point to the wrong location.

**Symptom:** `deploy.sh` constructs `us-central1-docker.pkg.dev/my-project/repo` but images are at `us-central1-docker.pkg.dev/my-project-dev/repo`. Image tagging succeeds (because the secret has the right path) but Cloud Run can't find the image.

**Fix:** Use a single `DOCKER_REGISTRY` secret for all scripts. Never construct the path from a project name.

## Gotcha #4: Version-Bump Direct Push Fails with enforce_admins

With `enforce_admins: true` on branch protection, CI automation cannot `git push origin main` directly — even with a valid PAT.

**Fix:** PR-based version bump flow:
1. Create `release/vX.Y.Z` branch
2. Open PR via GitHub API
3. Merge PR via GitHub API
4. Create tag via GitHub API (not `git tag` + `git push`)
5. Clean up release branch

Also add skip guards to prevent cascading empty releases:
- Skip `release: v*` commits (infinite loop prevention)
- Skip `Sync:` commits (sync-back merges)
- Skip `docs:` commits (documentation-only)
- Skip when `## Unreleased` section is empty

## Gotcha #5: Woodpecker depends_on Blocks on Non-Matching Workflows

If workflow A `depends_on: [B]` and workflow B has `branch: development` but the pipeline runs on staging, workflow A is blocked — Woodpecker doesn't skip non-matching dependencies.

**Symptom:** Smoke test never triggers on staging because it depends on `build` which only runs on development.

**Fix:** Inline dependent steps into each workflow rather than using cross-workflow dependencies:
```yaml
# build.yaml (development only): build → deploy → smoke-test → notify
# deploy.yaml (staging/main only): deploy → smoke-test → notify
```

## Gotcha #6: CI Should Not Re-Run on Promotion Branches

When a feature branch merges to development, CI already passed. Running CI again on the development push wastes time and agent capacity. Same for staging and main.

**Fix:** CI workflow excludes all promotion branches:
```yaml
when:
  - event: push
    branch:
      exclude: [development, staging, main]
```

Remove `depends_on: [ci]` from promote and deploy workflows — they'll hang if CI doesn't trigger.

## Gotcha #7: Binary PDF Merge Conflicts on Stacked PRs

Auto-generated PDFs (from `RELEASE_NOTES.md` → `docs/pdf/RELEASE_NOTES.pdf`) cause binary merge conflicts on every rebase when stacking PRs with squash merges.

**Fix:** Per the [develop-canonical PDF generation pattern](develop-canonical-pdf-generation.md), only generate PDFs on feature branches. Add a branch guard to the pre-commit hook:

```bash
CURRENT_BRANCH=$(git rev-parse --abbrev-ref HEAD)
if [ "$CURRENT_BRANCH" != "development" ] && [ "$CURRENT_BRANCH" != "staging" ] && [ "$CURRENT_BRANCH" != "main" ]; then
  # PDF generation here
fi
```

## Gotcha #8: Spot VM Kills Long Docker Builds

GCP spot VMs can be reclaimed at any time. A 15-minute Docker build has a significant chance of being killed mid-build. The Woodpecker pipeline shows status "killed" with deploy and notify steps skipped.

**Fix:** Baked base image (Gotcha #1 pattern). Keep builds under 60 seconds so they complete before spot reclamation is likely.

## Deployment Architecture

```
feature/* → development → staging → main
              ↓              ↓         ↓
            build          deploy    deploy
            deploy         smoke     smoke
            smoke          promote   version-bump
            promote                  sync-back
```

- **Feature branches:** CI only (test + lint + release notes check)
- **Development:** Docker build → deploy → smoke test → promote to staging
- **Staging:** Deploy (tag existing image) → smoke test → promote PR to main (manual)
- **Main:** Deploy → smoke test → version bump (PR-based) → tag → sync-back
