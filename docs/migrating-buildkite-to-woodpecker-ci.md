# Migrating from Buildkite to Self-Hosted Woodpecker CI

**Author:** Peregrine Technology Systems
**Date:** 2026-03-21
**Status:** In progress — first pipeline migrated, remaining 21 in queue

## Why We Moved

We migrated 22 projects from GitHub Actions to Buildkite in early 2026 to decouple CI/CD execution from Microsoft/GitHub infrastructure. Buildkite worked well technically but introduced two new problems:

1. **Plan-level concurrency limits** — Buildkite's free tier throttles concurrent jobs regardless of how many self-hosted agents you run. Our GCP agents sat idle while Buildkite refused to dispatch work. Upgrading costs $30/month with no guarantee the price won't change.

2. **SaaS vendor dependency** — Replacing one vendor dependency (GitHub Actions) with another (Buildkite SaaS) didn't achieve the decoupling goal. Buildkite controls pricing, API access, and feature availability.

**Woodpecker CI** (Apache-2.0, community fork of Drone) eliminates both: self-hosted server with zero concurrency limits, no SaaS dependency, and the option to fork if the project changes direction.

## Architecture

```
GitHub (webhooks) → Woodpecker Server (DigitalOcean droplet, $6/mo)
                         ↓ gRPC :9000
                    GCP Agents (existing MIG, spot VMs, scale-to-zero)
                         ↓ SSH
                    DigitalOcean Deploy Targets (existing droplets)
```

- **Server**: 1 vCPU / 1 GB DO droplet, Docker Compose, SQLite, Caddy for HTTPS
- **Agents**: Existing GCP e2-standard-4 spot VMs with local backend (host execution, no containers)
- **Secrets**: Woodpecker built-in secrets (repo/org level) + GCP Secret Manager for agent bootstrap

## What We Kept from Buildkite

| Component | Status |
|-----------|--------|
| GCP MIG + spot VMs | Unchanged |
| Packer custom image | Added Woodpecker agent binary |
| Deploy scripts (rsync + atomic symlink) | Unchanged |
| GCS cache bucket | Unchanged |
| Auto-merge GitHub Actions workflow | Unchanged |

## What Changed

| Before (Buildkite) | After (Woodpecker) |
|---------------------|---------------------|
| Buildkite SaaS ($30/mo) | DO droplet ($6/mo) |
| `.buildkite/pipeline.yml` | `.woodpecker/*.yaml` |
| Dynamic pipeline upload (`buildkite-agent pipeline upload`) | Multiple workflow files with `when` conditions |
| Environment hook (146 lines, GCS self-refresh) | Woodpecker built-in secrets |
| `buildkite-agent meta-data set/get` | Shared workspace files |
| `BUILDKITE_SOURCE` | `CI_PIPELINE_EVENT` |
| Buildkite API for scaler | Woodpecker REST API for scaler |

## Setup Guide

### 1. Woodpecker Server (DigitalOcean)

Smallest possible droplet. The server only coordinates — agents do the work.

```bash
# Create droplet
doctl compute droplet create woodpecker-ci \
  --region nyc1 --size s-1vcpu-1gb \
  --image ubuntu-24-04-x64 --ssh-keys <key-id>

# Create reserved IP (survives droplet replacement)
doctl compute reserved-ip create --region nyc1
doctl compute reserved-ip-action assign <ip> <droplet-id>
```

**Docker Compose** (`/opt/woodpecker/docker-compose.yml`):

```yaml
services:
  woodpecker-server:
    image: woodpeckerci/woodpecker-server:v3.13.0  # Pin version!
    ports:
      - "8000:8000"   # HTTP (behind Caddy)
      - "9000:9000"   # gRPC for agents
    volumes:
      - woodpecker-data:/var/lib/woodpecker/
    env_file:
      - .env
    restart: always

  caddy:
    image: caddy:2
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile:ro
      - caddy-data:/data
    depends_on:
      - woodpecker-server
    restart: always
```

**Gotcha: Pin the server version.** The `latest` tag on Docker Hub was 14 months stale (v2.8.3). We installed agent v3.13.0 and got `GRPC version mismatch` errors. Always pin to a specific version and match server + agent versions.

### 2. GitHub OAuth2 App

Woodpecker requires an **OAuth2 App**, NOT a GitHub App.

1. GitHub → Settings → Developer Settings → OAuth Apps → New
2. Callback URL: `https://your-domain/authorize`
3. Set `WOODPECKER_GITHUB_CLIENT` and `WOODPECKER_GITHUB_SECRET` in `.env`

**Gotcha: GitHub App will not work.** The Woodpecker docs explicitly warn about this. OAuth2 only.

### 3. GCP Agents

Install the agent binary in your Packer image alongside the existing Buildkite agent (run both during transition):

```bash
# In Packer provisioner
WPVER=$(curl -s https://api.github.com/repos/woodpecker-ci/woodpecker/releases/latest | jq -r .tag_name | sed 's/^v//')
curl -fsSL "https://github.com/woodpecker-ci/woodpecker/releases/download/v$WPVER/woodpecker-agent_linux_amd64.tar.gz" | tar xz -C /usr/local/bin/
```

**Agent config injection**: Store server URL and agent secret in GCP Secret Manager. A systemd oneshot service fetches them on boot and writes `/etc/woodpecker/agent.env` before the agent service starts.

**Gotcha: Use file provisioners for systemd units in Packer.** HCL heredocs fight with shell variable interpolation (`${VAR}` is treated as HCL interpolation). Create separate `.service` files and use `provisioner "file"` blocks instead.

**Gotcha: Agent config permission.** The agent runs as `buildkite-agent` user but tries to persist config to `/etc/woodpecker/agent.conf`. This fails with permission denied but is non-fatal — the agent still connects.

### 4. Pipeline Conversion

Woodpecker uses `.woodpecker/` directory with separate YAML files per workflow. Key syntax differences from Buildkite:

```yaml
# Woodpecker v3 syntax
labels:
  platform: linux
  backend: local          # Host execution (no containers)

when:
  - event: push
    branch: main

steps:
  - name: deploy
    image: bash            # Required even with local backend
    environment:
      DEPLOY_SSH_KEY:
        from_secret: deploy_ssh_key   # v3 secret syntax
    commands:
      - ./scripts/deploy.sh
```

**Gotcha: `.woodpecker/` treats ALL contents as pipeline configs.** If you put scripts in `.woodpecker/scripts/`, Woodpecker tries to parse the directory as a pipeline file and fails with `"is a folder not a file use Dir(..)"`. Put helper scripts outside `.woodpecker/` (we use `scripts/woodpecker/`).

**Gotcha: `secrets:` shorthand is deprecated in v3.** Use `environment` with `from_secret` instead:

```yaml
# WRONG (v2 syntax, causes error in v3)
secrets: [deploy_ssh_key]

# CORRECT (v3 syntax)
environment:
  DEPLOY_SSH_KEY:
    from_secret: deploy_ssh_key
```

**Gotcha: No dynamic pipeline upload.** Buildkite's `buildkite-agent pipeline upload` pattern (generating steps at runtime) doesn't exist in Woodpecker. Use multiple workflow files with `when` conditions instead. For branch-based routing, create separate `.yaml` files:

```
.woodpecker/
  deploy.yaml              # when: event=push, branch=main
  validate.yaml            # when: event=push, branch!=main
  security-scheduled.yaml  # when: event=cron
```

### 5. Bridging Buildkite Scripts

During migration, we kept existing Buildkite deploy scripts and created thin wrappers:

```bash
#!/bin/bash
# scripts/woodpecker/deploy.sh — wrapper for .buildkite/scripts/deploy.sh

export SSH_USER="$DEPLOY_TARGET_USER"
export SERVER_HOST="$DEPLOY_TARGET_HOST"

# Shim buildkite-agent meta-data with workspace files
export BACKUP_TIMESTAMP_FILE="${CI_WORKSPACE:-.}/.backup_timestamp"
buildkite-agent() {
  case "$1" in
    meta-data)
      case "$2" in
        set) echo "$4" > "$BACKUP_TIMESTAMP_FILE" ;;
        get) cat "$BACKUP_TIMESTAMP_FILE" 2>/dev/null || echo "" ;;
      esac ;;
  esac
}
export -f buildkite-agent

exec .buildkite/scripts/deploy.sh
```

This lets you validate the Woodpecker pipeline without rewriting deploy logic. Remove the wrappers and Buildkite scripts after full migration.

### 6. Activating Repos

Repos are activated via the Woodpecker UI or API:

```bash
# Get GitHub repo ID
REPO_ID=$(gh api repos/owner/repo --jq '.id')

# Activate in Woodpecker (note: query params, not body)
curl -X POST "https://your-server/api/repos?forge_remote_id=$REPO_ID&forge_id=1" \
  -H "Authorization: Bearer $TOKEN"

# Add secrets
curl -X POST "https://your-server/api/repos/$WP_REPO_ID/secrets" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"name":"deploy_ssh_key","value":"...","events":["push","cron"]}'
```

**Gotcha: Repo activation uses query parameters.** The v3 API rejects `forge_remote_id` in the request body with `"No forge_remote_id provided"`. Pass it as a query parameter instead.

## Gotchas Summary

| Issue | Solution |
|-------|----------|
| Docker Hub `latest` tag is stale | Pin server version: `woodpeckerci/woodpecker-server:v3.13.0` |
| GRPC version mismatch | Match server and agent versions exactly |
| GitHub App doesn't work | Use OAuth2 App only |
| Scripts in `.woodpecker/` break parsing | Put scripts outside `.woodpecker/` directory |
| `secrets:` shorthand deprecated (v3) | Use `environment:` with `from_secret:` |
| No dynamic pipeline upload | Use multiple workflow files with `when` conditions |
| `buildkite-agent meta-data` not available | Use shared workspace files |
| `BUILDKITE_SOURCE` not available | Use `CI_PIPELINE_EVENT` (`push`, `cron`, `pull_request`) |
| Repo activation API rejects body | Use query parameters for `forge_remote_id` |
| HCL heredocs eat `${}` in Packer | Use file provisioners for systemd units |
| SQLite migration fails (v2→v3) | Fix volume permissions: `chmod -R 777` on data dir |
| Agent can't write `/etc/woodpecker/agent.conf` | Non-fatal — agent still connects, ignore the error |

## Cost Comparison

| | Buildkite | Woodpecker |
|---|---|---|
| Control plane | $30/mo (SaaS) | $6/mo (DO droplet) |
| Agents | ~$0.04/hr (GCP spot) | Same |
| Concurrency | Plan-limited | Unlimited |
| Vendor dependency | Buildkite SaaS | None (Apache-2.0) |

## Migration Checklist

Per project:
- [ ] Create `.woodpecker/*.yaml` pipeline files
- [ ] Create wrapper scripts in `scripts/woodpecker/` (if reusing Buildkite scripts)
- [ ] Activate repo in Woodpecker UI/API
- [ ] Add secrets (deploy keys, target hosts)
- [ ] Verify: push triggers build → deploy → validate
- [ ] Remove `.buildkite/` directory
- [ ] Deactivate pipeline in Buildkite

Infrastructure (once):
- [ ] Deploy Woodpecker server on DO droplet
- [ ] Create GitHub OAuth2 App
- [ ] Add Woodpecker agent to Packer image
- [ ] Store agent config in GCP Secret Manager
- [ ] Adapt custom scaler to poll Woodpecker API
- [ ] Cancel Buildkite subscription

## Related Resources

- [Woodpecker CI Documentation](https://woodpecker-ci.org/docs/intro)
- [Peregrine Woodpecker fork](https://github.com/Peregrine-Technology-Systems/woodpecker)
- [Server deployment config](https://github.com/Peregrine-Technology-Systems/woodpecker-server)
- [Previous: Migrating GitHub Actions to Buildkite](./migrating-github-actions-to-buildkite-gcp.md)
