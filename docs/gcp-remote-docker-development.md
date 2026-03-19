# GCP Remote Docker Development with Persistent Layer Cache

A practical guide based on building a cloud development environment for a Docker-based penetration testing platform. This covers the GCP VM setup, persistent disk configuration for Docker layer caching, efficient code sync, auto-idle shutdown, and every gotcha we hit along the way.

## Why Remote Docker Builds?

Local Docker builds on macOS hit several walls: ARM-to-x86 cross-compilation is slow, large security tool images (OWASP ZAP, Nuclei, sqlmap, texlive) eat disk space, and full DAST scans need Linux-native networking. Running Docker builds on a GCP VM solves all of these:

- **Native x86 builds** — no QEMU emulation, no `--platform linux/amd64` overhead
- **Persistent layer cache** — Docker data-root on a persistent disk survives VM stop/delete
- **Cheap compute** — e2-standard-4 at ~$0.134/hr, only pay when actively developing
- **Isolation** — dev costs in a dedicated GCP project, separate from production and CI
- **Keep coding locally** — Claude Code and your editor stay on your laptop, only builds go remote

## Architecture Overview

```
MacOS (local)                          GCP (remote)
+----------------------------+         +-----------------------------------+
| Claude Code / Editor       |         | e2-standard-4 VM                  |
| Source code (git)          |   SSH   |   Docker daemon                   |
| ./cloud/dev CLI dispatcher |-------->|     data-root: /mnt/data/docker   |
|                            |  tar    |   BuildKit builder                |
|                            |-------->|     cache: /mnt/data/buildkit     |
+----------------------------+         +-----------------------------------+
                                              |
                                       +------+------+
                                       | 200GB pd-standard              |
                                       |   /mnt/data/                   |
                                       |   +-- docker/ (layer cache)    |
                                       |   +-- buildkit/ (buildx cache) |
                                       |   +-- results/ (scan output)   |
                                       |   +-- reports/ (generated PDF) |
                                       +--------------------------------+
```

Key properties:
- Source code lives on your laptop — you edit locally, sync to VM only for builds
- Docker daemon stores all layers on the persistent disk, not the VM boot disk
- Stopping or deleting the VM preserves all cached layers and scan results
- A single CLI dispatcher (`./cloud/dev`) handles start, stop, build, scan, SSH, and results

## Step 1: GCP Project Setup

Use a dedicated project to isolate dev costs from production and CI infrastructure:

```bash
# Create project
gcloud projects create peregrine-pentest-dev --name="Pentest Dev"
gcloud billing projects link peregrine-pentest-dev --billing-account=YOUR_BILLING_ID

# Enable Compute API
gcloud config set project peregrine-pentest-dev
gcloud services enable compute.googleapis.com
```

**Why a separate project?** Cost tracking becomes trivial — one line item per project in the billing dashboard. No tagging, no label filters, no shared-project cost allocation spreadsheets.

## Step 2: Create the Persistent Data Disk

Create the disk before the VM so it can be attached at creation time:

```bash
gcloud compute disks create pentest-dev-data \
  --size=200GB \
  --type=pd-standard \
  --zone=us-central1-a \
  --project=peregrine-pentest-dev
```

**Why 200GB?** GCP throttles IOPS and throughput on disks under 200GB. At 100GB you get performance warnings in the console and noticeably slower Docker pulls. 200GB pd-standard costs $8/month and eliminates the warnings entirely.

**Why pd-standard instead of pd-balanced?** For Docker layer cache workloads the sequential read/write performance is sufficient. pd-balanced costs 2x for IOPS you won't use during builds.

## Step 3: Create the VM

```bash
gcloud compute instances create pentest-dev \
  --machine-type=e2-standard-4 \
  --zone=us-central1-a \
  --image-family=ubuntu-2404-lts-amd64 \
  --image-project=ubuntu-os-cloud \
  --boot-disk-size=20GB \
  --boot-disk-type=pd-standard \
  --disk=name=pentest-dev-data,device-name=pentest-data,mode=rw,auto-delete=no \
  --project=peregrine-pentest-dev \
  --metadata=startup-script='#!/bin/bash
    # Mount data disk if not already mounted
    if ! mountpoint -q /mnt/data; then
      mkdir -p /mnt/data
      # Format only if not already formatted
      if ! blkid /dev/disk/by-id/google-pentest-data; then
        mkfs.ext4 -F /dev/disk/by-id/google-pentest-data
      fi
      mount /dev/disk/by-id/google-pentest-data /mnt/data
      # Ensure fstab entry
      if ! grep -q google-pentest-data /etc/fstab; then
        echo "/dev/disk/by-id/google-pentest-data /mnt/data ext4 defaults,nofail 0 2" >> /etc/fstab
      fi
    fi
  '
```

**Gotcha: Use `/dev/disk/by-id/google-{device-name}` not `/dev/sdb`.** Device paths like `/dev/sdb` are not stable across VM restarts — the kernel can assign a different letter each boot. The `by-id` path uses the `device-name` you set with `--disk` and is guaranteed stable.

**Key flag: `auto-delete=no`.** This ensures the data disk survives when you delete the VM. Without it, GCP deletes the disk along with the VM, destroying your entire layer cache.

## Step 4: Configure Docker Data Root on Persistent Disk

After the VM is running, SSH in and move Docker's storage to the persistent disk:

```bash
# SSH into the VM
gcloud compute ssh pentest-dev --zone=us-central1-a --project=peregrine-pentest-dev

# Install Docker
sudo apt-get update
sudo apt-get install -y docker.io docker-buildx

# Stop Docker
sudo systemctl stop docker docker.socket

# Create Docker data directory on persistent disk
sudo mkdir -p /mnt/data/docker

# Configure Docker to use persistent disk
sudo tee /etc/docker/daemon.json << 'EOF'
{
  "data-root": "/mnt/data/docker"
}
EOF

# Start Docker
sudo systemctl start docker

# Add your user to docker group
sudo usermod -aG docker $USER
```

After this, all Docker images, containers, volumes, and build cache live on the persistent disk. You can stop the VM, delete it, create a new one, point `data-root` at the same disk, and all your cached layers are back instantly.

### Verify It Works

```bash
# Check data-root
docker info | grep "Docker Root Dir"
# Should show: Docker Root Dir: /mnt/data/docker

# Pull a test image
docker pull ubuntu:24.04

# Verify it's on the persistent disk
ls /mnt/data/docker/overlay2/
# Should show layer directories
```

## Step 5: Configure BuildKit Cache on Persistent Disk

Docker BuildKit (used by `docker buildx`) maintains its own cache separate from Docker's layer cache. Point it at the persistent disk too:

```bash
# Create BuildKit cache directory
sudo mkdir -p /mnt/data/buildkit
sudo chmod 777 /mnt/data/buildkit

# Create a buildx builder that uses the persistent cache
docker buildx create \
  --name persistent-builder \
  --driver docker-container \
  --driver-opt="env.BUILDKIT_CACHE_DIR=/mnt/data/buildkit" \
  --use
```

**Gotcha: BuildKit cache directory permissions.** The buildx container runs as a non-root user internally. If the cache directory is owned by root with default permissions (755), BuildKit silently falls back to an in-container cache that disappears when the builder is recreated. Use `chmod 777` on the cache directory.

**Gotcha: Stale builder state after recreation.** If you `docker buildx rm persistent-builder` and recreate it, the old builder's cached errors can persist in Docker's metadata. Always remove and recreate together:

```bash
docker buildx rm persistent-builder 2>/dev/null || true
docker buildx create \
  --name persistent-builder \
  --driver docker-container \
  --use
```

## Step 6: CLI Dispatcher

Create a single entry point for all remote dev operations. This goes in your project repo at `./cloud/dev`:

```bash
#!/usr/bin/env bash
set -euo pipefail

PROJECT="peregrine-pentest-dev"
ZONE="us-central1-a"
VM="pentest-dev"

cmd_start() {
  echo "Starting VM..."
  gcloud compute instances start "$VM" --zone="$ZONE" --project="$PROJECT"
  echo "Waiting for SSH..."
  gcloud compute ssh "$VM" --zone="$ZONE" --project="$PROJECT" \
    --command="echo 'VM ready'" \
    --strict-host-key-checking=no
}

cmd_stop() {
  echo "Stopping VM..."
  gcloud compute instances stop "$VM" --zone="$ZONE" --project="$PROJECT"
}

cmd_ssh() {
  gcloud compute ssh "$VM" --zone="$ZONE" --project="$PROJECT" \
    --strict-host-key-checking=no
}

cmd_sync() {
  echo "Syncing code to VM..."
  local tar_opts=()
  if [ "${1:-}" = "--incremental" ] && [ -f .last-sync ]; then
    tar_opts+=(--newer-mtime="$(cat .last-sync)")
    echo "Incremental sync (files changed since $(cat .last-sync))"
  fi

  tar czf - "${tar_opts[@]}" \
    --exclude='.git' \
    --exclude='node_modules' \
    --exclude='.env' \
    --exclude='tmp' \
    --exclude='log' \
    . | gcloud compute ssh "$VM" --zone="$ZONE" --project="$PROJECT" \
        --strict-host-key-checking=no \
        --command="mkdir -p ~/project && cd ~/project && tar xzf -"

  date -u +"%Y-%m-%d %H:%M:%S" > .last-sync
  echo "Sync complete"
}

cmd_build() {
  cmd_sync --incremental
  echo "Building Docker image..."
  gcloud compute ssh "$VM" --zone="$ZONE" --project="$PROJECT" \
    --strict-host-key-checking=no \
    --command="cd ~/project && docker buildx build \
      --builder persistent-builder \
      --cache-from type=local,src=/mnt/data/buildkit \
      --cache-to type=local,dest=/mnt/data/buildkit,mode=max \
      -f docker/Dockerfile \
      -t pentest-platform:dev \
      --load ."
}

cmd_scan() {
  local target_url="${1:?Usage: ./cloud/dev scan <url>}"
  echo "Running scan against ${target_url}..."
  # Copy .env to VM temporarily
  gcloud compute scp .env "$VM":~/project/.env.tmp \
    --zone="$ZONE" --project="$PROJECT" \
    --strict-host-key-checking=no
  gcloud compute ssh "$VM" --zone="$ZONE" --project="$PROJECT" \
    --strict-host-key-checking=no \
    --command="cd ~/project && \
      mv .env.tmp /tmp/.env.scan && \
      docker run --rm \
        --env-file /tmp/.env.scan \
        -v /mnt/data/results:/app/storage \
        pentest-platform:dev \
        bundle exec rake scan:run TARGET_URLS='[\"${target_url}\"]' && \
      rm -f /tmp/.env.scan"
}

cmd_results() {
  echo "Fetching results..."
  gcloud compute scp --recurse \
    "$VM":/mnt/data/results/ ./results/ \
    --zone="$ZONE" --project="$PROJECT" \
    --strict-host-key-checking=no
  echo "Results downloaded to ./results/"
}

case "${1:-help}" in
  start)   cmd_start ;;
  stop)    cmd_stop ;;
  ssh)     cmd_ssh ;;
  sync)    cmd_sync "${2:-}" ;;
  build)   cmd_build ;;
  scan)    cmd_scan "${2:-}" ;;
  results) cmd_results ;;
  help)
    echo "Usage: ./cloud/dev <command>"
    echo "Commands: start, stop, ssh, sync, build, scan <url>, results"
    ;;
  *)
    echo "Unknown command: $1"
    exit 1
    ;;
esac
```

Make it executable: `chmod +x ./cloud/dev`

### Usage

```bash
./cloud/dev start              # Boot the VM
./cloud/dev build              # Sync code + Docker build
./cloud/dev scan https://target.example.com
./cloud/dev results            # Download scan results locally
./cloud/dev stop               # Shut down (cache preserved)
```

## Step 7: Efficient Code Sync with Differential Tar

The CLI dispatcher uses `tar --newer-mtime` for incremental syncs. On the first sync or after a clean, it sends everything. On subsequent syncs, it only tars files modified since the last sync:

```bash
# Full sync: ~15 seconds for a typical Rails project
tar czf - --exclude='.git' --exclude='.env' . | ssh vm "tar xzf -"

# Incremental sync: ~2 seconds for a few changed files
tar czf - --newer-mtime="2026-03-18 14:30:00" --exclude='.git' --exclude='.env' . | ssh vm "tar xzf -"
```

**Why tar over rsync?** For this use case, `tar --newer-mtime` through an SSH pipe is faster than rsync for small changesets because it avoids rsync's file-list negotiation round trip. For large changesets or binary files, rsync would be better — but during active development you're typically changing a handful of source files at a time.

**Fallback:** If the incremental sync misses files (clock skew, timezone issues), delete `.last-sync` and the next sync does a full transfer:

```bash
rm .last-sync
./cloud/dev sync
```

**Critical: `.env` is excluded from sync tarballs.** Credentials never travel in the code sync. The `.env` file is copied separately at scan time via `gcloud compute scp` and placed in `/tmp` on the VM where it's deleted after the scan completes.

## Step 8: Auto-Idle Shutdown

Prevent forgotten VMs from running up costs with a cron job that checks for activity every 5 minutes:

```bash
#!/bin/bash
# /usr/local/bin/auto-idle-shutdown.sh

IDLE_THRESHOLD=1800  # 30 minutes in seconds

# Check for active SSH sessions
if who | grep -q pts; then
  exit 0
fi

# Check for running Docker containers (excluding paused)
if [ "$(docker ps -q --filter 'status=running' 2>/dev/null | wc -l)" -gt 0 ]; then
  exit 0
fi

# Check for active BuildKit processes
if docker buildx ls 2>/dev/null | grep -q running; then
  exit 0
fi

# Check uptime (don't shutdown during boot)
uptime_seconds=$(awk '{print int($1)}' /proc/uptime)
if [ "$uptime_seconds" -lt "$IDLE_THRESHOLD" ]; then
  exit 0
fi

# Check last activity timestamp
LAST_ACTIVITY="/tmp/last-dev-activity"
if [ -f "$LAST_ACTIVITY" ]; then
  last_ts=$(stat -c %Y "$LAST_ACTIVITY" 2>/dev/null || echo 0)
  now_ts=$(date +%s)
  idle_seconds=$((now_ts - last_ts))
  if [ "$idle_seconds" -lt "$IDLE_THRESHOLD" ]; then
    exit 0
  fi
fi

# No activity — shut down
logger "auto-idle-shutdown: No activity for ${IDLE_THRESHOLD}s, shutting down"
/sbin/shutdown -h now
```

Install the cron job:

```bash
# Install the script
sudo cp auto-idle-shutdown.sh /usr/local/bin/
sudo chmod +x /usr/local/bin/auto-idle-shutdown.sh

# Run every 5 minutes
echo "*/5 * * * * root /usr/local/bin/auto-idle-shutdown.sh" | sudo tee /etc/cron.d/auto-idle-shutdown
```

**How it works:** The script checks for SSH sessions, running Docker containers, and active BuildKit processes. If none are found and the VM has been idle for 30 minutes, it triggers a shutdown. Since the data disk persists, no cache or results are lost.

## Step 9: Handling gcloud SSH Gotchas

Several `gcloud compute ssh` and `gcloud compute scp` behaviors differ from raw SSH/SCP in ways that bite you during scripting.

**Gotcha: `gcloud compute scp` does NOT accept raw SSH options after `--`.** Unlike raw `scp`, you cannot pass `-o StrictHostKeyChecking=no` after `--`. Use the built-in flag instead:

```bash
# WRONG — silently ignored or errors
gcloud compute scp file vm:~/path -- -o StrictHostKeyChecking=no

# RIGHT
gcloud compute scp file vm:~/path --strict-host-key-checking=no
```

**Gotcha: `gcloud compute ssh --command` buffers all output.** When you run a long command via `--command`, gcloud buffers all stdout and stderr until the command completes. You see nothing for minutes, then everything at once. This is particularly painful for Docker builds where you want to watch layer progress.

**Workaround for streaming:** For long-running commands, use interactive SSH instead:

```bash
# Buffered (no output until complete)
gcloud compute ssh vm --command="docker build -t app ."

# Streaming (output as it happens)
gcloud compute ssh vm -- "docker build -t app ."
# Or use the dispatcher's SSH command and run manually
./cloud/dev ssh
# Then inside the VM:
cd ~/project && docker build -t app .
```

## Step 10: Dev Environment vs Production Configuration

When running scans on the dev VM, you want reports written to the persistent disk, not uploaded to GCS. This requires adjusting environment variables:

```bash
# In your .env for dev scans, comment out or remove:
# GCS_BUCKET=your-prod-bucket
# GOOGLE_CLOUD_PROJECT=your-prod-project

# Reports will fall back to local storage
STORAGE_TYPE=local
STORAGE_PATH=/app/storage
```

**Gotcha: If `GCS_BUCKET` is set in `.env`, the app tries to upload to GCS.** Even on a dev VM, the storage service follows the environment. Strip GCS-related variables from the dev `.env` so reports go to local storage (which is the persistent disk via the Docker volume mount).

### texlive Packages for PDF Generation

**Gotcha: `texlive-xetex` alone is not enough for pandoc PDF generation.** The penetration test report templates use LaTeX packages like `soul.sty` that live in `texlive-latex-extra`. Without it, pandoc fails with `File 'soul.sty' not found`:

```dockerfile
# WRONG — PDF generation fails
RUN apt-get install -y texlive-xetex

# RIGHT — includes soul.sty and other common packages
RUN apt-get install -y texlive-xetex texlive-latex-extra texlive-fonts-recommended
```

## Cost Management

### Running Costs

| Component | Monthly Cost | Notes |
|-----------|-------------|-------|
| e2-standard-4 (4 hrs/day, 5 days/week) | ~$12 | 87 hrs/month at $0.134/hr |
| 200GB pd-standard | $8 | Always-on, even when VM stopped |
| Network egress | ~$1 | Downloading results and reports |
| **Total** | **~$21** | |

### Cost Reduction Strategies

- **Auto-idle shutdown** prevents forgotten VMs (see Step 8)
- **Stop, don't delete** the VM between sessions — boot is faster than recreating
- **Dedicated project** makes cost tracking trivial — one line in billing
- **pd-standard over pd-balanced** saves 50% on disk cost for this workload

### If You Forget to Stop the VM

An e2-standard-4 running 24/7 costs ~$97/month. The auto-idle shutdown script catches most cases, but as a safety net, set a billing alert:

```bash
# Set a budget alert at $30/month
gcloud billing budgets create \
  --billing-account=YOUR_BILLING_ID \
  --display-name="Pentest dev VM" \
  --budget-amount=30 \
  --threshold-rule=percent=80 \
  --threshold-rule=percent=100 \
  --filter-projects=projects/peregrine-pentest-dev
```

## Security Considerations

### Credential Handling

- `.env` is excluded from code sync tarballs — never travels with the source
- `.env` is copied to the VM only at scan time via `gcloud compute scp`
- On the VM, `.env` goes to `/tmp` and is deleted after the scan completes
- No GCS credentials needed for dev — reports go to local persistent disk storage

### SSH Access

- SSH access is IAM-gated through `gcloud compute ssh` — no manually managed SSH keys
- The default-allow-ssh firewall rule permits SSH from anywhere by default
- For tighter security, restrict to your IP range:

```bash
# Replace default-allow-ssh with restricted rule
gcloud compute firewall-rules update default-allow-ssh \
  --source-ranges="YOUR_IP/32" \
  --project=peregrine-pentest-dev
```

### Data at Rest

- The persistent disk contains scan results, which include vulnerability details about target systems
- GCP encrypts all disks at rest by default (AES-256, Google-managed keys)
- For additional protection, use Customer-Managed Encryption Keys (CMEK):

```bash
gcloud compute disks create pentest-dev-data \
  --kms-key=projects/PROJECT/locations/LOCATION/keyRings/RING/cryptoKeys/KEY \
  --size=200GB --type=pd-standard --zone=us-central1-a
```

## Buildkite Integration: Two Independent Build Paths

The dev VM and Buildkite CI are completely independent systems that share no infrastructure:

```
Developer workflow (interactive):
  Local code --> tar sync --> Dev VM --> docker buildx build --> scan --> results

CI workflow (automated):
  git push --> Buildkite --> GCP agent VM --> docker build --> push to Artifact Registry
```

### Key Differences

| Aspect | Dev VM | Buildkite CI |
|--------|--------|-------------|
| **Purpose** | Interactive development and testing | Automated builds on push |
| **GCP Project** | peregrine-pentest-dev | ci-runners-de |
| **BuildKit cache** | Local disk (`/mnt/data/buildkit`) | Registry-based (`type=registry`) |
| **VM lifecycle** | Long-lived, stopped when idle | Ephemeral, destroyed after build |
| **Docker images** | Local only (`--load`) | Pushed to Artifact Registry |
| **Secrets** | `.env` file via scp | GCP Secret Manager (auto-discovery) |

### Registry-Based Cache for CI vs Local Cache for Dev

Buildkite agents are ephemeral — they scale to zero and are destroyed after idle. A local disk cache would be lost every time. Instead, CI uses registry-based BuildKit cache:

```bash
# CI build (Buildkite agent)
docker buildx build \
  --cache-from type=registry,ref=REGION-docker.pkg.dev/PROJECT/REPO/cache \
  --cache-to type=registry,ref=REGION-docker.pkg.dev/PROJECT/REPO/cache,mode=max \
  --push -t REGION-docker.pkg.dev/PROJECT/REPO/image:tag .
```

```bash
# Dev build (persistent VM)
docker buildx build \
  --cache-from type=local,src=/mnt/data/buildkit \
  --cache-to type=local,dest=/mnt/data/buildkit,mode=max \
  --load -t pentest-platform:dev .
```

### Cross-Project Artifact Registry Access

If Buildkite agents in `ci-runners-de` need to push/pull images from a registry in another project:

```bash
# Grant the Buildkite agent SA access to the registry
gcloud artifacts repositories add-iam-policy-binding REPO_NAME \
  --project=REGISTRY_PROJECT \
  --location=REGION \
  --member="serviceAccount:buildkite-agent@ci-runners-de.iam.gserviceaccount.com" \
  --role="roles/artifactregistry.writer"
```

### Secrets Convention

Buildkite pipeline secrets use GCP Secret Manager with auto-discovery via naming convention:

```bash
# Create: {pipeline-slug}--{secret-name}
echo -n "value" | gcloud secrets create \
  my-pipeline--my-secret-name \
  --data-file=- --project=ci-runners-de

# Grant agent access
gcloud secrets add-iam-policy-binding my-pipeline--my-secret-name \
  --member="serviceAccount:buildkite-agent@ci-runners-de.iam.gserviceaccount.com" \
  --role="roles/secretmanager.secretAccessor" \
  --project=ci-runners-de
```

Auto-exported as `MY_SECRET_NAME` on deploy steps. Dashes become underscores, uppercased.

## What Worked Well

A summary of design decisions that paid off:

1. **Persistent disk separate from VM** — stop the VM nightly, delete and recreate it when Ubuntu needs upgrading, cache and results survive everything. The disk outlives any number of VMs.

2. **Docker data-root on persistent disk** — a single `daemon.json` change gives you a fully persistent layer cache. No registry pulls after the first build. A 2GB Docker image that takes 3 minutes to pull is instant on rebuild.

3. **Differential tar sync** — `tar --newer-mtime` with a timestamp file makes code sync near-instant during active development. Change one Ruby file, sync takes under 2 seconds.

4. **Auto-idle shutdown** — the 5-minute cron check catches every case: forgot to stop the VM, walked away from the keyboard, meeting ran long. Combined with the billing alert, runaway costs are impossible.

5. **Dedicated GCP project** — no cost allocation debates, no shared quota concerns, no accidental resource deletion by teammates working in the same project.

6. **CLI dispatcher** — a single `./cloud/dev` script means you never have to remember gcloud flags, zone names, or project IDs. Every operation is one command.

## Checklist

Before your first remote build:

- [ ] Dedicated GCP project created with billing linked
- [ ] Compute API enabled
- [ ] 200GB pd-standard persistent disk created (separate from VM)
- [ ] VM created with data disk attached (`auto-delete=no`)
- [ ] Data disk mounted at `/mnt/data` with fstab entry
- [ ] Docker installed with `data-root` set to `/mnt/data/docker`
- [ ] BuildKit cache directory created at `/mnt/data/buildkit` with `chmod 777`
- [ ] Buildx builder created using persistent cache directory
- [ ] Auto-idle shutdown script installed as cron job
- [ ] CLI dispatcher (`./cloud/dev`) in project repo and executable
- [ ] `.env` excluded from sync tarball excludes list
- [ ] Dev `.env` stripped of GCS/production variables
- [ ] Billing alert set on the dev project
- [ ] `texlive-latex-extra` included in Dockerfile for PDF generation
- [ ] Firewall rule reviewed (restrict SSH source range if needed)

## Summary of Gotchas

| Issue | Symptom | Fix |
|-------|---------|-----|
| Device path `/dev/sdb` not stable | Disk mounts to wrong path or fails after reboot | Use `/dev/disk/by-id/google-{device-name}` |
| BuildKit cache dir permissions | Cache not persisted, rebuilds from scratch every time | `chmod 777 /mnt/data/buildkit` — buildx container runs non-root |
| Stale buildx builder after recreate | Old errors persist, builds fail with cached error | `docker buildx rm` then recreate — always do both together |
| `gcloud compute scp` SSH options | `-o StrictHostKeyChecking=no` silently ignored | Use `--strict-host-key-checking=no` flag instead |
| `gcloud compute ssh --command` buffering | No output for minutes during Docker build | Use `gcloud compute ssh vm -- "command"` for streaming |
| GCS_BUCKET set in dev `.env` | Reports uploaded to production GCS instead of local disk | Remove GCS_BUCKET and GOOGLE_CLOUD_PROJECT from dev `.env` |
| `texlive-xetex` missing packages | `File 'soul.sty' not found` during PDF generation | Install `texlive-latex-extra` alongside `texlive-xetex` |
| Disk under 200GB | GCP console performance warnings, slower I/O | Use 200GB minimum for pd-standard |
| VM deleted with `auto-delete=yes` | Persistent disk destroyed, all cache and results lost | Always set `auto-delete=no` on data disk |
| Forgot to stop VM | $97/month instead of $12/month | Auto-idle shutdown cron + billing alert at $30 |
