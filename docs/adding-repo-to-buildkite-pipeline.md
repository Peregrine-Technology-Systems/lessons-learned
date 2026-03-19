# Adding a New Repo to GCP-Based Buildkite Pipeline

A practical guide based on adding the `web-app-penetration-test` repo to an existing Buildkite + GCP Elastic CI Stack. This assumes you already have the infrastructure from [Migrating GitHub Actions to Buildkite with GCP](migrating-github-actions-to-buildkite-gcp.md) — this covers only the per-repo onboarding steps and every gotcha we hit.

## Prerequisites

- Buildkite organization with GCP Elastic CI Stack running
- GCP project `ci-runners-de` with Secret Manager and agent SA configured
- Buildkite API token with `write_pipelines` scope (or UI access)

## Step 1: Create the Pipeline Definition in Your Repo

Add `.buildkite/pipeline.yml` to your repo:

```yaml
agents:
  queue: gcp

steps:
  - label: ":rspec: Test"
    command: |
      bundle install --jobs 4
      bundle exec rspec
    plugins:
      - docker#v5.11.0:
          image: "ruby:3.2.2"

  - label: ":rubocop: Lint"
    command: |
      bundle install --jobs 4
      bundle exec rubocop
    plugins:
      - docker#v5.11.0:
          image: "ruby:3.2.2"

  - wait

  - label: ":docker: Build"
    command: |
      docker build -f docker/Dockerfile -t pentest-platform .

  - wait

  - label: ":rocket: Promote"
    command: ".buildkite/scripts/promote.sh"
    branches: "development"
```

**Gotcha: The top-level `agents: queue: gcp` is required.** Without it, jobs sit in `platform_limited` state forever. The elastic stack only picks up jobs tagged for its queue.

## Step 2: Create the Pipeline in Buildkite

Via API:

```bash
curl -X POST "https://api.buildkite.com/v2/organizations/YOUR_ORG/pipelines" \
  -H "Authorization: Bearer $BUILDKITE_API_TOKEN" \
  -d '{
    "name": "web-app-penetration-test",
    "repository": "git@github.com:Peregrine-Technology-Systems/web-app-penetration-test.git",
    "default_branch": "development",
    "cluster_id": "YOUR_CLUSTER_UUID",
    "steps": [
      {
        "type": "script",
        "name": ":pipeline: Upload",
        "command": "buildkite-agent pipeline upload",
        "agent_query_rules": ["queue=gcp"]
      }
    ]
  }'
```

**Gotcha: The repo URL must be SSH format (`git@github.com:`) not HTTPS.** Buildkite agents clone via SSH using deploy keys. HTTPS URLs fail silently or prompt for credentials that don't exist on the agent.

**Gotcha: The upload step needs `agent_query_rules: ["queue=gcp"]` too.** This is the bootstrap step that reads your `pipeline.yml` — if it doesn't target the right queue, nothing runs.

## Step 3: Create the GitHub Webhook

In the repo's GitHub settings (Settings > Webhooks > Add webhook):

| Field | Value |
|-------|-------|
| Payload URL | `https://webhook.buildkite.com/deliver/YOUR_WEBHOOK_TOKEN` |
| Content type | `application/json` |
| Events | Push, Pull Request, Deployment |

Get the webhook URL from the Buildkite pipeline settings page after creating the pipeline.

## Step 4: Create Pipeline Secrets in GCP Secret Manager

Secrets follow the `{pipeline-slug}--{secret-name}` naming convention:

```bash
# Example: Docker registry credentials
echo -n "your-token" | gcloud secrets create \
  web-app-penetration-test--docker-registry-token \
  --data-file=- --project=ci-runners-de

# Example: Deploy SSH key
gcloud secrets create \
  web-app-penetration-test--deploy-ssh-key \
  --data-file=/path/to/id_ed25519 --project=ci-runners-de
```

**The environment hook auto-discovers secrets matching `{pipeline-slug}--*`.** No hook changes are needed for standard secrets — they are exported as uppercased env vars with dashes converted to underscores (e.g., `DOCKER_REGISTRY_TOKEN`).

**Only update the environment hook's `case` statement if you need deploy SSH setup or app-specific secret logic** (e.g., writing a key to `~/.ssh/` before clone).

## Step 5: Grant Agent SA Access to Secrets and Cross-Project Resources

```bash
# Grant access to each secret
gcloud secrets add-iam-policy-binding \
  web-app-penetration-test--deploy-ssh-key \
  --member="serviceAccount:buildkite-agent@ci-runners-de.iam.gserviceaccount.com" \
  --role="roles/secretmanager.secretAccessor" \
  --project=ci-runners-de

# If pushing to Artifact Registry in another project:
gcloud projects add-iam-policy-binding TARGET_PROJECT_ID \
  --member="serviceAccount:buildkite-agent@ci-runners-de.iam.gserviceaccount.com" \
  --role="roles/artifactregistry.writer"
```

## Step 6: Add Deploy Key (If Needed)

If the repo is under a different GitHub org or user than your existing deploy keys cover:

```bash
# Generate a new key pair
ssh-keygen -t ed25519 -f /tmp/buildkite-deploy-key -N "" -C "buildkite-pentest"

# Add public key to GitHub repo: Settings > Deploy keys > Add
# Title: "Buildkite CI"  |  Check "Allow write access" if promotion creates PRs

# Store private key in Secret Manager
gcloud secrets create web-app-penetration-test--github-deploy-key \
  --data-file=/tmp/buildkite-deploy-key --project=ci-runners-de

# Clean up local copy
rm /tmp/buildkite-deploy-key /tmp/buildkite-deploy-key.pub
```

## Step 7: Update Branch Protection

In GitHub repo settings (Settings > Branches > Branch protection rules):

1. Edit the rule for `development` (or your target branch)
2. Under "Require status checks to pass before merging," add the Buildkite check name
3. The check name format is: `buildkite/PIPELINE_SLUG` (e.g., `buildkite/web-app-penetration-test`)
4. Remove any old GitHub Actions checks that are no longer running

## Gotchas Reference

| # | Gotcha | Impact |
|---|--------|--------|
| 1 | Missing `agents: queue: gcp` in pipeline.yml | Jobs stuck in `platform_limited` — never picked up |
| 2 | HTTPS repo URL instead of SSH | Clone fails — agents use deploy keys, not tokens |
| 3 | Using `ruby:3.2.2-slim` Docker image | `bundle install` fails on native gems (nokogiri, pg) — needs build-essential, use full `ruby:3.2.2` |
| 4 | `GITHUB_TOKEN` in promotion workflows | Can't create cross-repo PRs or push — use a PAT or deploy key with write access |
| 5 | Editing environment hook for standard secrets | Unnecessary — auto-discovery handles `{pipeline-slug}--*` pattern without changes |
| 6 | Missing deploy key for new org/user | Clone fails with `Permission denied (publickey)` — each repo needs its own deploy key |
| 7 | Upload step missing queue tag | Bootstrap step runs nowhere — add `agent_query_rules` to the initial upload step |
| 8 | `ghcr.io/cli/cli` Docker image requires auth | `docker pull` fails — GHCR requires authentication even for public images | Use `curl`/`jq` against the GitHub API instead of depending on `gh` CLI in containers |
| 9 | Using Docker plugin for promotion steps | Unnecessary complexity, image pull issues | Use a shell script (`.buildkite/scripts/promote.sh`) that calls GitHub API directly with curl. Runs on the agent, no Docker plugin needed |
| 10 | `ruby:3.2.2-slim` missing native compilation deps | `bundle install` fails — no build-essential, no libsqlite3-dev | Use full `ruby:3.2.2` image instead of slim for projects with native gem extensions |

## Verification Checklist

After completing all steps, verify the pipeline works:

```bash
# Trigger a test build
git commit --allow-empty -m "chore: test Buildkite pipeline" && git push

# Check Buildkite dashboard for the build
# Verify: upload step runs > pipeline steps appear > all pass

# Verify secrets are accessible (add a debug step temporarily)
# steps:
#   - label: "debug"
#     command: "echo $DOCKER_REGISTRY_TOKEN | wc -c"  # length only, never print secrets
```

Once the build goes green, remove any debug steps and you're done.
