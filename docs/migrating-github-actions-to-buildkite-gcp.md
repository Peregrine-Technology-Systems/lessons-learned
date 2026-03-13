# Migrating from GitHub Actions to Buildkite with GCP Elastic Runners

A practical guide based on a real migration of production infrastructure deployments. This covers the GCP setup, Buildkite pipeline design, and every gotcha we hit along the way.

## Why Move?

GitHub Actions is convenient but comes with trade-offs: opaque runner infrastructure, limited debugging when things go wrong, and vendor lock-in to GitHub's ecosystem. Buildkite with GCP Elastic CI Stack gives you:

- **Your infrastructure, your control** — agents run on your GCP VMs
- **Scale to zero** — pay only for build minutes (~$0.25/month at 100 builds)
- **Secrets in GCP Secret Manager** — IAM-controlled, audit-logged, never in a web UI
- **Spot instances** — 56% cheaper than on-demand
- **Shadow mode** — run both systems in parallel before cutting over

## Architecture Overview

```
GitHub (repos, webhooks)
    │
    ▼ webhook on push
Buildkite SaaS (scheduling, logs, dashboard)
    ▲
    │ HTTPS poll (outbound only)
GCP Elastic CI Stack
    │ Cloud NAT (no external IPs on VMs)
    │
    ▼ SSH deploy
Your servers (DigitalOcean, AWS, etc.)
```

Key properties:
- Source code cloned from GitHub by GCP agents — never touches Buildkite servers
- Secrets stored in GCP Secret Manager — never sent to Buildkite
- Agents poll Buildkite outbound over HTTPS — no inbound firewall rules needed
- VMs are ephemeral — destroyed after idle, natural key rotation

## Step 1: GCP Project Setup

```bash
# Create project (name must be globally unique)
gcloud projects create my-ci-runners --name="CI Runners"
gcloud billing projects link my-ci-runners --billing-account=YOUR_BILLING_ID

# Enable required APIs
gcloud config set project my-ci-runners
gcloud services enable \
  compute.googleapis.com \
  secretmanager.googleapis.com \
  cloudfunctions.googleapis.com \
  cloudscheduler.googleapis.com \
  cloudbuild.googleapis.com \
  run.googleapis.com

# Create Terraform state bucket
gsutil mb -l us-central1 gs://my-ci-runners-terraform-state
gsutil versioning set on gs://my-ci-runners-terraform-state
```

**Gotcha:** GCP project IDs are globally unique. If `ci-runners` is taken, try `ci-runners-yourorg`.

## Step 2: Store Secrets

```bash
# Buildkite agent token
echo -n "bkua_your_token_here" | \
  gcloud secrets create buildkite-agent-token --data-file=-

# SSH deploy key for your target servers
gcloud secrets create deploy-ssh-key \
  --data-file=/path/to/id_ed25519

# Deploy target config (per-project)
echo '{"ssh_user":"root","server_host":"x.x.x.x"}' | \
  gcloud secrets create deploy-target-myapp --data-file=-

# GitHub deploy key (for cloning private repos)
gcloud secrets create github-deploy-key \
  --data-file=/path/to/github_deploy_key
```

**Gotcha: Trailing newlines matter.** When `gcloud secrets versions access` writes a key to a file, it preserves the exact bytes. If your key was stored without a trailing newline, OpenSSH will reject it with `error in libcrypto`. Always append a newline after writing:

```bash
gcloud secrets versions access latest --secret="deploy-ssh-key" > ~/.ssh/id_ed25519
echo "" >> ~/.ssh/id_ed25519  # OpenSSH requires trailing newline
chmod 600 ~/.ssh/id_ed25519
```

## Step 3: Terraform Configuration

```hcl
# main.tf
terraform {
  required_version = ">= 1.0"
  backend "gcs" {
    bucket = "my-ci-runners-terraform-state"
    prefix = "ci-infrastructure"
  }
  required_providers {
    google = {
      source  = "hashicorp/google"
      version = "~> 5.0"
    }
  }
}

provider "google" {
  project = var.project_id
  region  = var.region
}

module "buildkite" {
  source  = "buildkite/elastic-ci-stack-for-gcp/buildkite"
  version = "~> 0.4.0"

  project_id                   = var.project_id
  buildkite_organization_slug  = var.buildkite_org_slug
  buildkite_agent_token_secret = "projects/${var.project_id}/secrets/buildkite-agent-token/versions/latest"

  machine_type      = "e2-standard-2"  # 2 vCPU, 8 GB — enough for most CI
  min_size          = 0                 # Scale to zero when idle
  max_size          = 3                 # Max concurrent builds
  root_disk_size_gb = 50
  root_disk_type    = "pd-balanced"

  region = var.region
  zones  = ["${var.region}-a", "${var.region}-b"]

  buildkite_queue      = "gcp"
  enable_secret_access = true
}
```

**Gotcha: The `buildkite_agent_token_secret` format.** The module validates this with a regex expecting `projects/PROJECT_ID/secrets/SECRET_NAME/versions/VERSION`. Using `google_secret_manager_secret.name` returns a numeric project ID which also works, but using the string project ID is clearer.

**Gotcha: Terraform variable naming.** Don't name your variable `buildkite_agent_token` — it conflicts with the module's internal variable of the same name, causing both the direct token and the secret reference to be set (which errors). Use something like `buildkite_agent_token_value`.

**Gotcha: Zones must be explicit.** If you don't pass `zones`, the module uses `null` which fails the `length()` validation. Always pass explicit zones.

### Phased Apply

The module has dependencies that can't all resolve in a single `terraform apply`:

```bash
terraform init
terraform apply -target=google_secret_manager_secret.buildkite_agent_token
terraform apply -target=module.buildkite.module.iam
terraform apply  # Full apply now succeeds
```

## Step 4: Agent Environment Hook

This is the glue — it runs before every build step and injects per-pipeline secrets:

```bash
#!/bin/bash
# /etc/buildkite-agent/hooks/environment
set -euo pipefail

# GitHub deploy key for cloning repos
setup_github_ssh() {
  mkdir -p ~/.ssh
  chmod 700 ~/.ssh
  gcloud secrets versions access latest --secret="github-deploy-key" > ~/.ssh/github_deploy_key
  echo "" >> ~/.ssh/github_deploy_key  # trailing newline
  chmod 600 ~/.ssh/github_deploy_key
  ssh-keyscan -H github.com >> ~/.ssh/known_hosts 2>/dev/null || true

  cat >> ~/.ssh/config <<'SSHCONFIG'
Host github.com
  IdentityFile ~/.ssh/github_deploy_key
  StrictHostKeyChecking no
SSHCONFIG
  chmod 600 ~/.ssh/config
}

# Server deploy key
setup_deploy_ssh() {
  gcloud secrets versions access latest --secret="deploy-ssh-key" > ~/.ssh/id_ed25519
  echo "" >> ~/.ssh/id_ed25519  # trailing newline
  chmod 600 ~/.ssh/id_ed25519
  ssh-keyscan -H "${SERVER_HOST}" >> ~/.ssh/known_hosts 2>/dev/null || true
}

setup_github_ssh

# Per-pipeline secret injection
case "${BUILDKITE_PIPELINE_SLUG}" in
  my-nginx-project)
    TARGET_JSON=$(gcloud secrets versions access latest --secret="deploy-target-myapp")
    export SSH_USER=$(echo "$TARGET_JSON" | jq -r '.ssh_user')
    export SERVER_HOST=$(echo "$TARGET_JSON" | jq -r '.server_host')
    setup_deploy_ssh
    ;;
  another-project)
    # Different server, different secrets
    TARGET_JSON=$(gcloud secrets versions access latest --secret="deploy-target-other")
    export SSH_USER=$(echo "$TARGET_JSON" | jq -r '.ssh_user')
    export SERVER_HOST=$(echo "$TARGET_JSON" | jq -r '.server_host')
    setup_deploy_ssh
    ;;
  *)
    # CI-only pipelines: no deploy secrets needed
    ;;
esac
```

**Gotcha: Hook deployment.** The environment hook must exist on every agent VM at `/etc/buildkite-agent/hooks/environment`. With elastic agents, new VMs are created and destroyed dynamically — you can't deploy hooks manually. Options:

1. **GCS bucket + startup script (recommended)** — upload the hook to a GCS bucket, download it in the VM startup script. This survives scale-to-zero and lets you update hooks without rebuilding images:

```bash
# Upload hook to GCS
gsutil mb -l us-central1 gs://my-ci-runners-agent-hooks
gsutil cp agent/hooks/environment gs://my-ci-runners-agent-hooks/environment

# Grant read access to agent service account
gsutil iam ch "serviceAccount:buildkite-agent@my-project.iam.gserviceaccount.com:objectViewer" \
  gs://my-ci-runners-agent-hooks

# In your startup script:
HOOKS_DIR="/etc/buildkite-agent/hooks"
gsutil cp "gs://my-ci-runners-agent-hooks/environment" "${HOOKS_DIR}/environment"
chmod +x "${HOOKS_DIR}/environment"
chown buildkite-agent:buildkite-agent "${HOOKS_DIR}/environment"
```

2. **Bake into a custom GCE image** — build with Packer, set as the instance template image. Faster cold starts but requires image rebuilds to update hooks.

3. **Deploy manually to each VM** — quick for testing but **does not survive scaling**. When VMs are destroyed and recreated (scale-to-zero), the hook is gone.

## Step 5: Pipeline Design

### Use Dynamic Pipeline Upload (Not `if:` Conditions)

**Gotcha: `if:` conditions show "broken" steps.** If you use `if:` to conditionally skip steps (e.g., `if: "build.branch == 'main'"`), steps where the condition evaluates to false show as **"broken"** in the Buildkite UI — not "skipped". This makes passing builds look like failures, which is confusing and can trigger false alerts.

**Fix:** Use a setup step that dynamically uploads only the relevant pipeline steps based on the branch. This way skipped steps never appear at all.

```yaml
# .buildkite/pipeline.yml — bootstrap only
steps:
  - label: ":pipeline: Setup pipeline"
    command: ".buildkite/scripts/setup-pipeline.sh"
    agents:
      queue: "gcp"
```

```bash
#!/bin/bash
# .buildkite/scripts/setup-pipeline.sh
set -euo pipefail

if [ "${BUILDKITE_BRANCH}" = "main" ]; then
  cat <<'PIPELINE' | buildkite-agent pipeline upload
steps:
  - label: ":rocket: Deploy"
    command: ".buildkite/scripts/deploy.sh"
    agents:
      queue: "gcp"
    retry:
      automatic:
        - signal_reason: agent_stop
          limit: 2

  - wait

  - label: ":mag: Validate"
    command: ".buildkite/scripts/validate.sh"
    agents:
      queue: "gcp"
    timeout_in_minutes: 10

  - wait: ~
    continue_on_failure: true

  - label: ":rotating_light: Rollback"
    command: ".buildkite/scripts/rollback.sh"
    if: "build.state == 'failing'"
    agents:
      queue: "gcp"
PIPELINE

else
  cat <<'PIPELINE' | buildkite-agent pipeline upload
steps:
  - label: ":white_check_mark: Validate configs"
    command: ".buildkite/scripts/validate-configs.sh"
    agents:
      queue: "gcp"
PIPELINE

fi
```

Note: the Rollback step still uses `if: "build.state == 'failing'"` — this is fine because it only appears on main builds where it's always relevant. It will show as "broken" only if the deploy passes (which is the expected outcome), but since it's a rollback step, seeing it dimmed is less confusing than seeing "Validate configs: broken" on a main branch deploy.

**Gotcha: `branches` and `if` conflict.** Buildkite rejects steps that have both `branches:` and `if:`. Use only `if:` for conditional logic.

### Skip Redundant Builds

When a feature branch merges to main, the commit was already tested. Skip the redundant deploy:

```bash
#!/bin/bash
# .buildkite/scripts/check-already-tested.sh
set -euo pipefail

COMMIT="${BUILDKITE_COMMIT}"
BRANCH="${BUILDKITE_BRANCH}"
PIPELINE="${BUILDKITE_PIPELINE_SLUG}"
ORG="${BUILDKITE_ORGANIZATION_SLUG}"

# Only skip on main
[ "${BRANCH}" != "main" ] && exit 0

# Check if this commit already passed
PRIOR=$(curl -sf \
  -H "Authorization: Bearer ${BUILDKITE_AGENT_ACCESS_TOKEN:-}" \
  "https://api.buildkite.com/v2/organizations/${ORG}/pipelines/${PIPELINE}/builds?commit=${COMMIT}&state=passed&per_page=1" \
  2>/dev/null || echo "[]")

COUNT=$(echo "$PRIOR" | python3 -c "import json,sys; print(len(json.load(sys.stdin)))" 2>/dev/null || echo "0")

if [ "$COUNT" -gt 0 ]; then
  echo "Already tested — skipping"
  echo "steps: []" | buildkite-agent pipeline upload
  exit 0
fi

# Also check parent commits (for merge commits)
for parent in $(git rev-parse HEAD^@ 2>/dev/null); do
  PARENT_BUILD=$(curl -sf \
    -H "Authorization: Bearer ${BUILDKITE_AGENT_ACCESS_TOKEN:-}" \
    "https://api.buildkite.com/v2/organizations/${ORG}/pipelines/${PIPELINE}/builds?commit=${parent}&state=passed&per_page=1" \
    2>/dev/null || echo "[]")

  PARENT_COUNT=$(echo "$PARENT_BUILD" | python3 -c "import json,sys; print(len(json.load(sys.stdin)))" 2>/dev/null || echo "0")

  if [ "$PARENT_COUNT" -gt 0 ]; then
    echo "Parent commit already tested — skipping"
    echo "steps: []" | buildkite-agent pipeline upload
    exit 0
  fi
done
```

## Step 6: SSH Reliability

This is where most of the real-world pain lives. Your GCP agents SSH to your deploy targets over the public internet through Cloud NAT. Things go wrong.

### Use Retry Wrappers

```bash
SSH_OPTS="-o ConnectTimeout=10 -o ServerAliveInterval=30 -o StrictHostKeyChecking=no"

ssh_retry() {
  local max_attempts=3
  local attempt=1
  while [ $attempt -le $max_attempts ]; do
    if ssh ${SSH_OPTS} "$@"; then
      return 0
    fi
    echo "SSH attempt ${attempt}/${max_attempts} failed, retrying in 5s..."
    sleep 5
    attempt=$((attempt + 1))
  done
  echo "SSH failed after ${max_attempts} attempts"
  return 1
}

# Use throughout your deploy script
ssh_retry "${SSH_USER}@${SERVER_HOST}" << 'EOF'
  sudo systemctl restart nginx
EOF
```

### Cloud NAT IP Is Not Static

GCP Cloud NAT with `AUTO_ONLY` uses ephemeral IPs that can change. You **cannot** whitelist a specific IP on your target server. Instead:

- Use key-only SSH authentication (no password auth)
- Use fail2ban for brute-force protection
- **Do not** use UFW rate limiting on SSH (see below)

### Target Server Firewall Rules (Critical)

**Never use `ufw limit ssh` on servers that receive CI deployments.** UFW's rate limit (6 connections per 30 seconds) causes `Connection refused` for CI agents that make multiple SSH connections in quick succession (keyscan + deploy + validate).

```bash
# WRONG — will block CI agents
ufw limit ssh

# RIGHT — fail2ban handles brute-force instead
ufw allow 22/tcp
```

**Never use `ufw --force reset` in deployment scripts.** It drops ALL rules instantly, including SSH access. If the deploy fails after the reset, you're locked out and need console access to recover.

```bash
# WRONG — drops SSH access during deploy
ufw --force reset
ufw allow ssh
ufw allow 80/tcp
# ... if script fails here, SSH is gone

# RIGHT — idempotent, never drops existing rules
ensure_rule() {
  if ! ufw status | grep -qF "$*" 2>/dev/null; then
    ufw $*
  fi
}
ensure_rule "allow 22/tcp"
ensure_rule "allow 80/tcp"
ensure_rule "allow 443/tcp"
```

### systemd Socket Activation

Modern Ubuntu uses `ssh.socket` for SSH, which has its own rate limiting separate from UFW:

```
TriggerLimitBurst=20    # default: 20 connections per 2 seconds
TriggerLimitIntervalSec=2s
```

If you see `Connection refused` despite UFW being open, increase the burst limit:

```bash
sudo mkdir -p /etc/systemd/system/ssh.socket.d
cat << 'EOF' | sudo tee /etc/systemd/system/ssh.socket.d/override.conf
[Socket]
TriggerLimitBurst=100
TriggerLimitIntervalSec=5s
EOF
sudo systemctl daemon-reload
sudo systemctl restart ssh.socket
```

## Step 7: Feature Branch Validation

For infrastructure projects (nginx configs, etc.), validate syntax without deploying:

```bash
#!/bin/bash
# .buildkite/scripts/validate-configs.sh
set -euo pipefail

# Install nginx for syntax checking
sudo apt-get update -qq
sudo apt-get install -y -qq nginx > /dev/null 2>&1

# Create dummy SSL certs (configs reference /etc/letsencrypt/...)
for domain in your.domain.com staging.domain.com; do
  cert_dir="/etc/letsencrypt/live/${domain}"
  sudo mkdir -p "${cert_dir}"
  sudo openssl req -x509 -nodes -newkey rsa:2048 \
    -keyout "${cert_dir}/privkey.pem" \
    -out "${cert_dir}/fullchain.pem" \
    -subj "/CN=${domain}" -days 1 2>/dev/null
  sudo cp "${cert_dir}/fullchain.pem" "${cert_dir}/chain.pem"
done

# Deploy configs and test
sudo cp configs/nginx/*.conf /etc/nginx/snippets/
for config in configs/nginx/site-*.conf; do
  sudo cp "$config" /etc/nginx/sites-available/
  sudo ln -sf "/etc/nginx/sites-available/$(basename $config)" /etc/nginx/sites-enabled/
done

sudo nginx -t
```

**Gotcha: GCP agent VMs are minimal.** They don't have `rsync`, `nc` (netcat), or other tools you might expect. Install what you need at the start of your scripts, or bake them into a custom image.

## Step 8: Running in Parallel (Shadow Mode)

Run Buildkite alongside GitHub Actions before cutting over:

1. **GitHub Actions remains primary** — branch protection rules still check GHA
2. **Buildkite runs in shadow mode** — same deploy, same server, results compared
3. **Both deploy the same configs** — safe for idempotent operations (nginx, static files)
4. **Buildkite commit status appears on GitHub** — visible alongside GHA checks

This lets you validate the entire pipeline without risk. When both consistently pass, switch branch protection to require Buildkite checks instead.

### GitHub Commit Status Integration

For Buildkite build results to appear on GitHub commits alongside GitHub Actions checks, you need the **Buildkite GitHub App** installed — not just the webhook.

**Gotcha: Webhook vs GitHub App.** If you created the pipeline via the API or connected it with a webhook, `publish_commit_status: true` is set but statuses **won't appear on GitHub**. The pipeline's `provider_id` will show `"github"` (webhook) instead of `"github_app"`. To fix:

1. Install the Buildkite GitHub App: **Buildkite → Organization Settings → Repository Providers → GitHub**
2. Authorize the app on GitHub and grant access to your repos
3. **Delete and recreate the pipeline** — existing pipelines don't switch providers automatically. The API doesn't support changing `provider_id` after creation.

```bash
# Delete old pipeline
curl -s -X DELETE -H "Authorization: Bearer ${TOKEN}" \
  "https://api.buildkite.com/v2/organizations/ORG/pipelines/my-pipeline"

# Recreate — will now use GitHub App if installed
curl -s -X POST -H "Authorization: Bearer ${TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{"name":"my-pipeline","repository":"git@github.com:org/repo.git",
       "default_branch":"main","cluster_id":"YOUR_CLUSTER_ID",
       "configuration":"steps:\n  - command: \"buildkite-agent pipeline upload\"\n    agents:\n      queue: \"gcp\""}' \
  "https://api.buildkite.com/v2/organizations/ORG/pipelines"
```

**Gotcha: Cluster ID required.** If your org has Clusters, the API returns `Cluster must be specified` without the `cluster_id` field. Get it with:

```bash
curl -s -H "Authorization: Bearer ${TOKEN}" \
  "https://api.buildkite.com/v2/organizations/ORG/clusters" | jq '.[].id'
```

## Step 9: Buildkite Clusters + Scale-to-Zero Fix

If your Buildkite organization has Clusters enabled (mandatory for some orgs), the Elastic CI Stack's built-in autoscaler **will not work**. It relies on a custom GCP metric (`custom.googleapis.com/buildkite/.../UnfinishedJobsCount`) written by a Go Cloud Function that queries the Buildkite metrics API — but the metrics API returns 401 for clustered orgs.

Without a fix, you're stuck paying for always-on VMs (~$49/month per e2-standard-2, or ~$147/month for 3 agents).

### The Fix: Custom Scaler Cloud Function

Bypass the broken autoscaler entirely with a lightweight Python Cloud Function that:
1. Polls the Buildkite REST API (which works fine with Clusters) for scheduled/running jobs
2. Directly resizes the MIG via the GCP Compute API

```python
# cloud-functions/scaler/main.py (simplified)
import json, os, urllib.request
import functions_framework
import google.auth
import google.auth.transport.requests

BUILDKITE_API_URL = "https://api.buildkite.com/v2"
COMPUTE_API_URL = "https://compute.googleapis.com/compute/v1"

def get_job_counts(org_slug, queue, token):
    """Count scheduled and running jobs for the given queue."""
    scheduled = running = 0
    for state in ("scheduled", "running"):
        url = f"{BUILDKITE_API_URL}/organizations/{org_slug}/builds?state={state}&per_page=100"
        req = urllib.request.Request(url, headers={"Authorization": f"Bearer {token}"})
        with urllib.request.urlopen(req, timeout=10) as resp:
            builds = json.loads(resp.read())
        for build in builds:
            for job in build.get("jobs", []):
                if f"queue={queue}" in job.get("agent_query_rules", []):
                    if job.get("state") == "scheduled":
                        scheduled += 1
                    elif job.get("state") in ("running", "assigned", "accepted"):
                        running += 1
    return scheduled, running

def resize_mig(project_id, region, mig_name, target_size, max_size):
    """Set MIG target size via REST API."""
    credentials, _ = google.auth.default(scopes=["https://www.googleapis.com/auth/compute"])
    credentials.refresh(google.auth.transport.requests.Request())

    size = max(0, min(target_size, max_size))

    # Check current size
    url = f"{COMPUTE_API_URL}/projects/{project_id}/regions/{region}/instanceGroupManagers/{mig_name}"
    req = urllib.request.Request(url, headers={"Authorization": f"Bearer {credentials.token}"})
    with urllib.request.urlopen(req, timeout=10) as resp:
        current = json.loads(resp.read())["targetSize"]

    if current == size:
        return size

    # Resize
    url += f"/resize?size={size}"
    req = urllib.request.Request(url, method="POST", headers={
        "Authorization": f"Bearer {credentials.token}",
        "Content-Type": "application/json",
    })
    urllib.request.urlopen(req, timeout=10)
    return size

@functions_framework.http
def scale(request):
    token = os.environ["BUILDKITE_API_TOKEN"]
    scheduled, running = get_job_counts(
        os.environ["BUILDKITE_ORG_SLUG"],
        os.environ["BUILDKITE_QUEUE"], token)
    total_demand = scheduled + running
    new_size = resize_mig(
        os.environ["GCP_PROJECT_ID"], os.environ["GCP_REGION"],
        os.environ["MIG_NAME"], total_demand,
        int(os.environ.get("MAX_SIZE", "3")))
    return json.dumps({"scheduled": scheduled, "running": running,
                       "mig_target_size": new_size}), 200
```

Requirements (`requirements.txt`):
```
functions-framework==3.*
google-auth==2.*
requests==2.*
```

### Terraform for the Scaler

Key resources needed:
1. **Service account** with `roles/compute.instanceAdmin.v1` (to resize MIG) and `roles/secretmanager.secretAccessor` (to read the Buildkite API token)
2. **Cloud Function** (Gen2/Python 3.12, 128Mi memory, 30s timeout)
3. **Cloud Scheduler** job (`* * * * *` — every 60 seconds)
4. **Cloud Run invoker IAM** — the scheduler's OIDC token must be allowed to invoke the function

```hcl
# Disable the module's built-in autoscaler
module "buildkite" {
  # ... existing config ...
  enable_autoscaling = false  # Our custom scaler replaces this
}

# Grant the scaler SA permission to invoke the function
resource "google_cloud_run_v2_service_iam_member" "scaler_invoker" {
  project  = var.project_id
  location = var.region
  name     = google_cloudfunctions2_function.scaler.service_config[0].service
  role     = "roles/run.invoker"
  member   = "serviceAccount:${google_service_account.scaler.email}"
}
```

**Gotcha: Cloud Functions Gen2 URLs.** Gen2 functions run on Cloud Run. The `google_cloudfunctions2_function.scaler.url` attribute gives the Cloud Functions URL, but Cloud Scheduler needs the Cloud Run URL for OIDC auth. Use `service_config[0].uri` instead:

```hcl
resource "google_cloud_scheduler_job" "scaler" {
  http_target {
    uri = google_cloudfunctions2_function.scaler.service_config[0].uri  # NOT .url
  }
}
```

**Gotcha: MIG autoscaler conflict.** You cannot manually resize a MIG that has a GCP autoscaler attached — you'll get HTTP 412 (Precondition Failed). You must set `enable_autoscaling = false` in the module and `terraform apply` to delete the old autoscaler before the custom scaler can work.

**Gotcha: Cloud Function memory.** Don't use `google-cloud-compute` (the full SDK) — it exceeds 128Mi on cold start. Use `google-auth` + `urllib.request` to call the Compute REST API directly. This keeps the function under 128Mi.

### Cost: $0.00/month

The scaler function runs within GCP's free tier:
- ~43,200 invocations/month (every 60s) — free tier covers 2 million
- 128Mi × <1s per invocation — free tier covers 400,000 GB-seconds

### Add Email Alerts

Use GCP Cloud Monitoring to alert when the scaler fails:

```hcl
resource "google_monitoring_notification_channel" "scaler_email" {
  display_name = "Scaler alert"
  type         = "email"
  labels = { email_address = "you@example.com" }
}

resource "google_monitoring_alert_policy" "scaler_errors" {
  display_name = "Buildkite scaler failures"
  combiner     = "OR"
  conditions {
    display_name = "Cloud Function execution errors"
    condition_threshold {
      filter          = "resource.type = \"cloud_function\" AND resource.labels.function_name = \"buildkite-scaler\" AND metric.type = \"cloudfunctions.googleapis.com/function/execution_count\" AND metric.labels.status != \"ok\""
      comparison      = "COMPARISON_GT"
      threshold_value = 0
      duration        = "0s"
      aggregations {
        alignment_period   = "300s"
        per_series_aligner = "ALIGN_COUNT"
      }
      trigger { count = 1 }
    }
  }
  notification_channels = [google_monitoring_notification_channel.scaler_email.name]
  alert_strategy { auto_close = "1800s" }
}
```

## Step 10: Create the Buildkite Pipeline

```bash
# Via API
curl -s -X POST \
  -H "Authorization: Bearer ${BUILDKITE_API_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "my-project",
    "repository": "git@github.com:org/my-project.git",
    "default_branch": "main",
    "configuration": "steps:\n  - command: \"buildkite-agent pipeline upload\"\n    label: \":pipeline:\"\n    agents:\n      queue: \"gcp\""
  }' \
  "https://api.buildkite.com/v2/organizations/YOUR_ORG/pipelines"
```

Then set up the GitHub webhook (Buildkite does this automatically if you connect your GitHub account in the dashboard).

## Cost Reality

At ~100 builds/month with 5-minute average duration on spot instances:

| Component | Monthly Cost |
|-----------|-------------|
| GCP Compute (spot, scale-to-zero) | ~$0.25 |
| GCP Secret Manager | ~$0.03 |
| Cloud Function (scaler) | $0.00 (free tier) |
| Cloud Scheduler | $0.00 (3 free jobs) |
| Cloud Monitoring alerts | $0.00 (free tier) |
| GCS (hook storage) | $0.00 (free tier) |
| Buildkite (Developer plan, 3 users) | $0.00 |
| **Total** | **~$0.28** |

Even at 1,000 builds/month, it's under $3.

**Without the custom scaler:** If you skip the scale-to-zero fix and set `min_size=1`, the always-on VM costs ~$49/month (on-demand) or ~$21/month (spot). With 3 agents always running (as the default module configures), that's ~$147/month — all for idle VMs.

## Checklist

Before your first build:

- [ ] GCP project created with billing
- [ ] APIs enabled (Compute, Secret Manager, Cloud Functions, Cloud Scheduler, Cloud Build, Cloud Run)
- [ ] Terraform state bucket created with versioning
- [ ] Buildkite agent token in Secret Manager
- [ ] Buildkite API token in Secret Manager (for custom scaler)
- [ ] SSH deploy key in Secret Manager (with trailing newline)
- [ ] GitHub deploy key in Secret Manager + added to repo
- [ ] Deploy target secrets in Secret Manager
- [ ] `terraform apply` completed (may need phased apply)
- [ ] Built-in autoscaler disabled (`enable_autoscaling = false`)
- [ ] Custom scaler Cloud Function deployed and tested
- [ ] Cloud Scheduler triggering scaler every 60s
- [ ] Email alert policy configured for scaler failures
- [ ] Environment hook uploaded to GCS bucket
- [ ] Startup script downloads hook from GCS on boot
- [ ] Agent service account has read access to hook bucket
- [ ] Target server firewall: `ufw allow 22/tcp` (not `limit`), fail2ban active
- [ ] Target server: `ssh.socket` TriggerLimitBurst increased if needed
- [ ] Buildkite GitHub App installed (for commit status reporting)
- [ ] Pipeline created in Buildkite (with GitHub App provider, not webhook)
- [ ] `.buildkite/pipeline.yml` and scripts in your repo
- [ ] Test build passes on feature branch
- [ ] Test build passes on main (full deploy)
- [ ] Verify scale-to-zero: VMs terminate when no builds are queued
- [ ] Verify scale-up: VMs start within ~90s when a build is triggered

## Summary of Gotchas

| Issue | Symptom | Fix |
|-------|---------|-----|
| Secret key missing trailing newline | `error in libcrypto` | `echo "" >> keyfile` after writing |
| `ufw limit ssh` | `Connection refused` from CI | Use `ufw allow 22/tcp` + fail2ban |
| `ufw --force reset` in deploy scripts | SSH lockout mid-deploy | Idempotent `ensure_rule` pattern |
| ssh.socket TriggerLimitBurst | Intermittent `Connection refused` | Override to 100 in systemd drop-in |
| Cloud NAT IP is ephemeral | Can't whitelist CI IP on target server | Key-only auth + fail2ban |
| Buildkite Clusters + metrics API | Autoscaler can't read queue depth, VMs run 24/7 | Custom scaler Cloud Function (see Step 9) |
| MIG resize returns HTTP 412 | `Precondition Failed` on resize call | Delete the GCP autoscaler first (`enable_autoscaling = false`) |
| Cloud Function OOM on cold start | `Memory limit of 128 MiB exceeded` | Use `google-auth` + REST API, not full `google-cloud-compute` SDK |
| Cloud Function auth fails | `request was not authenticated` | Grant `roles/run.invoker` to the scheduler's service account |
| Gen2 function URL mismatch | Scheduler calls wrong URL, gets 401 | Use `service_config[0].uri` (Cloud Run URL), not `.url` (CF URL) |
| Buildkite commit status missing | Pipeline passes but nothing shows on GitHub | Install GitHub App + recreate pipeline (webhook provider won't post statuses) |
| Pipeline API: cluster required | `Cluster must be specified` error | Include `cluster_id` in API request body |
| Hook missing after scale-to-zero | `SSH_USER: unbound variable` | Deploy hook via GCS bucket + startup script |
| `if:` conditions on skipped steps | Steps show as "broken" not "skipped" in UI | Use dynamic pipeline upload — only upload relevant steps per branch |
| `branches` + `if` in pipeline YAML | Pipeline upload rejected | Use only `if:` expressions |
| Agent VMs missing tools | `rsync: command not found` | Install in script or custom image |
| `nc` not available on GCP agents | Port checks fail | Use `curl` instead of `nc -zv` |
| HTTP 302 treated as failure | Validation rejects valid redirects | Accept 200, 301, 302, 404, 502, 503 |
| Terraform variable name collision | Conflicting agent token values | Don't name vars same as module internals |
| Terraform zones null | `length()` validation failure | Pass explicit zones list |
| Terraform phased apply needed | Count dependencies on apply-time values | Apply secrets, then IAM, then full |
