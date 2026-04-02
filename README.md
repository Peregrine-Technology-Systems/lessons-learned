# Lessons Learned

Practical guides from real-world infrastructure work. Each document covers a specific migration or implementation, including every gotcha encountered along the way.

These guides are written to be useful for both humans and AI coding assistants working on similar problems.

## Documents

| Guide | Description |
|-------|-------------|
| [Migrating GitHub Actions to Buildkite with GCP Elastic Runners](docs/migrating-github-actions-to-buildkite-gcp.md) | Complete guide to replacing GitHub Actions with Buildkite on GCP, including scale-to-zero, SSH reliability, firewall safety, and 22 documented gotchas |
| [GCP Remote Docker Development with Persistent Layer Cache](docs/gcp-remote-docker-development.md) | Cloud dev environment on GCP with persistent Docker layer cache, differential code sync, auto-idle shutdown, and 10 documented gotchas |
| [Adding a New Repo to GCP-Based Buildkite Pipeline](docs/adding-repo-to-buildkite-pipeline.md) | Per-repo onboarding to existing Buildkite + GCP infrastructure, including secrets, deploy keys, webhooks, and 7 documented gotchas |
| [Develop-Canonical PDF Generation Pattern](docs/develop-canonical-pdf-generation.md) | Auto-generate doc PDFs on feature branches only, track in git, flow forward via promotions without binary merge conflicts, and 6 documented gotchas |
| [Ruby + Sinatra + Docker on GCP Cloud Run](docs/ruby-sinatra-docker-cloud-run.md) | Baked base image pattern (45x faster builds), Sinatra 4 host authorization, Cloud Run IAM vs app auth, cross-project registry, version-bump PR flow, and 8 documented gotchas |
| [Automated Branch Promotion Pipeline](docs/automated-branch-promotion-pipeline.md) | Three-branch Git flow with conflict-free automated promotions, always-local merge branches (GitHub API ignores .gitattributes), cascade guards, shallow clone pitfalls, and 7 documented learnings |

## Contributing

Found something that should be added or corrected? Open an issue or PR.

## License

MIT
