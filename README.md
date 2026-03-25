# et-upcloud

UpCloud infrastructure skills for Claude Code — provision and deploy full-stack apps with the `upctl` CLI.

> **When to use et-upcloud:** Setting up servers, databases, object storage, and secret management on UpCloud. Deploying apps with Docker Compose + Caddy auto-SSL. For AWS/GCP/Azure, use other tooling.

## Workflow

```
  1. SETUP                          2. DEPLOY                       3. MANAGE
  /upcloud:setup                    /upcloud:deploy push            /upcloud:deploy status
  ┌──────────────────┐              ┌──────────────────┐            ┌────────────┐
  │ Create server    │              │ rsync code       │            │ Health     │
  │ Provision DB     │  ──config──▶ │ Inject secrets   │  ──live──▶│ Logs       │
  │ Setup secrets    │              │ Docker Compose   │            │ Rollback   │
  │ Generate config  │              │ Caddy auto-SSL   │            │ Secrets    │
  └──────────────────┘              └──────────────────┘            └────────────┘
       writes .deploy.json               reads .deploy.json
```

## Architecture

```
  ┌─────────────────────────────────────────────────────┐
  │ UpCloud Cloud Server (Docker + Caddy + Infisical)   │
  │                                                     │
  │  ┌─────────┐  ┌─────────┐  ┌──────────────┐       │
  │  │ Your App│  │  Caddy  │  │  Infisical   │       │
  │  │ :8080   │◀─│ :443    │  │  (secrets)   │       │
  │  └─────────┘  └─────────┘  └──────────────┘       │
  └─────────────────────────────────────────────────────┘
           │                              │
           ▼                              ▼
  ┌─────────────────┐          ┌───────────────────┐
  │ Managed         │          │ Object Storage    │
  │ PostgreSQL      │          │ (S3-compatible)   │
  │ + pgvector      │          │                   │
  │ + pg_cron       │          │                   │
  └─────────────────┘          └───────────────────┘
```

## Skills

| Skill | Command | Description |
| ----- | ------- | ----------- |
| **setup** | `/upcloud:setup` | Provision server, DB, storage, secrets, roles |
| **deploy** | `/upcloud:deploy push` | Sync code + rebuild containers |
| | `/upcloud:deploy migrate` | Run database migrations |
| | `/upcloud:deploy status` | Health check + container status |
| | `/upcloud:deploy logs` | Stream service logs |
| | `/upcloud:deploy rollback` | Revert to previous version |
| | `/upcloud:deploy secrets` | Manage secrets (list/add/update) |

## Secret Management — Low Risk Only

| Provider | Risk | Effort | Rotation | Audit Trail |
| -------- | ---- | ------ | -------- | ----------- |
| **Infisical** (self-hosted) | Low | Medium | Automatic | Full |
| **Docker Secrets** | Low | Low | Manual | None |
| **S3 Bundle** (encrypted) | Acceptable | Low | Manual | S3 logs |

Encrypted `.env` in containers is **never** supported — high risk, no rotation, no audit trail.

## Installation

### Prerequisites

1. **Claude Code** — [Install Claude Code](https://docs.anthropic.com/en/docs/claude-code/overview)
2. **upctl** — UpCloud CLI tool:
   ```bash
   # macOS
   brew tap UpCloudLtd/tap && brew install upcloud-cli

   # Linux (deb)
   # Download from https://github.com/UpCloudLtd/upcloud-cli/releases

   # Verify
   upctl version
   ```
3. **Authenticate upctl:**
   ```bash
   upctl account login --with-token
   # Or set: export UPCLOUD_TOKEN="your-token"
   ```

### Install the plugin

From inside Claude Code:

```bash
# Add marketplace
/plugin marketplace add Emerging-Tech-Visma/et-upcloud

# Install the plugin
/plugin install upcloud@et-upcloud
```

To pin a specific version:

```bash
/plugin marketplace add Emerging-Tech-Visma/et-upcloud@v1.2.0
```

### Verify installation

```bash
# In Claude Code, run:
/upcloud:setup
```

Claude should ask for your project name, zone, and features.

## Quick Start

### 1. Provision infrastructure

```
/upcloud:setup
```

Answer the prompts:
- **Project name**: `myapp`
- **Zone**: `fi-hel1` (default, Helsinki EU)
- **Server plan**: `2xCPU-4GB` (default)
- **Database**: Yes
- **Object storage**: No
- **Secret provider**: Infisical (recommended)

This creates the server, database, and secrets — then writes `.deploy.json` to your project.

### 2. Deploy your app

```
/upcloud:deploy push
```

Syncs code via rsync, injects secrets, rebuilds Docker containers, and runs a health check.

### 3. Check status

```
/upcloud:deploy status
```

Shows container health, endpoint status, and resource usage.

## File Structure

```
et-upcloud/
├── .claude-plugin/
│   └── marketplace.json              ← marketplace manifest
├── et-upcloud-plugin/                ← plugin directory
│   ├── .claude-plugin/
│   │   ├── plugin.json               ← plugin identity + version
│   │   └── settings.json             ← permissions
│   ├── CLAUDE.md                     ← instructions loaded when active
│   ├── commands/
│   │   ├── setup.md                  ← /setup shortcut
│   │   ├── deploy.md                 ← /deploy shortcut
│   │   └── server-status.md          ← /server-status shortcut
│   └── skills/
│       ├── upcloud-setup/
│       │   ├── SKILL.md              ← setup skill definition
│       │   ├── references/           ← provisioning playbooks
│       │   └── templates/            ← docker-compose, Caddyfile, etc.
│       └── upcloud-deploy/
│           ├── SKILL.md              ← deploy skill definition
│           └── references/           ← deploy, migrate, rollback playbooks
├── CLAUDE.md                         ← project overview
├── CHANGELOG.md
└── README.md
```

## Cost Estimate

Minimal viable setup per project:

| Resource | Plan | ~EUR/month |
| -------- | ---- | ---------- |
| Cloud Server | 2xCPU-4GB | ~22 |
| Managed PostgreSQL | 1xCPU-2GB-25GB | ~16 |
| Object Storage | 250GB | ~5 |
| **Total** | | **~43** |

Multiple projects can share one server and one PG instance (separate databases).

## Principles

1. **No secrets on disk** — everything through the configured provider
2. **No secrets in images** — injected at runtime only
3. **No deletes without approval** — always confirm destructive operations
4. **EU data residency** — `fi-hel1` default
5. **Least privilege** — three DB roles, scoped storage keys
6. **Idempotent** — safe to run setup twice
7. **Observable** — every deploy reports health

## License

MIT
