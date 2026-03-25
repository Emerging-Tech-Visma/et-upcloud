---
name: upcloud-deploy
description: >
  Deploy and manage applications on UpCloud servers. Pushes code via rsync, runs database
  migrations, builds and restarts Docker containers with Infisical secret injection, checks
  health endpoints, streams logs, manages rollbacks, and handles secrets. Reads .deploy.json
  for project configuration. Use this skill whenever the user wants to deploy code, push changes,
  check deployment status, view logs, roll back a deployment, run migrations, or manage secrets
  on UpCloud — even if they just say "deploy this", "push to prod", "check the server",
  "show me the logs", or "roll back".
---

# /upcloud:deploy — Continuous Deployment

Pushes code changes to UpCloud servers and manages running services. Reads `.deploy.json` to know where everything lives.

## Prerequisites

- `.deploy.json` exists in project root (created by `/upcloud:setup`)
- SSH access to the server (key-based, configured during setup)
- Secret backend configured (check `secrets.provider` in `.deploy.json`)

Always start by reading `.deploy.json`:
```bash
cat .deploy.json | jq '.'
```

## Secret Backend Detection

Read `secrets.provider` from `.deploy.json` to determine how secrets are injected:

| Provider | How to start containers | How to manage secrets |
| -------- | ---------------------- | -------------------- |
| `infisical` | `infisical run --env=prod -- docker compose up -d` | `infisical secrets set/list` |
| `docker-secrets` | `docker compose up -d` (secrets auto-mounted) | Edit files in `secrets_dir`, restart |
| `s3-bundle` | `docker compose up -d` (entrypoint fetches) | Re-encrypt + upload bundle, restart |

All commands below use `{START_CMD}` as a placeholder — substitute the correct command based on the provider.

## Commands

### push — Sync Code to Server

Read `references/deploy-push.md` for the full playbook.

Syncs the project directory to the server via rsync, then rebuilds and restarts containers:

```bash
# Read config
SERVER_IP=$(jq -r '.server.ip' .deploy.json)
PROJECT=$(jq -r '.project' .deploy.json)

# Sync code (fast incremental transfer)
rsync -avz --delete \
  --exclude '.git' \
  --exclude 'node_modules' \
  --exclude '.env' \
  --exclude '.deploy.json' \
  -e ssh \
  ./ root@${SERVER_IP}:/opt/${PROJECT}/

# Rebuild and restart on server (adapt to secrets provider)
PROVIDER=$(jq -r '.secrets.provider' .deploy.json)

# Infisical:
ssh root@${SERVER_IP} "cd /opt/${PROJECT} && infisical run --env=prod -- docker compose -f docker-compose.prod.yml up -d --build"

# Docker Secrets or S3 Bundle:
ssh root@${SERVER_IP} "cd /opt/${PROJECT} && docker compose -f docker-compose.prod.yml up -d --build"
```

After push, automatically run a health check.

### migrate — Run Database Migrations

Read `references/deploy-migrate.md` for the full playbook.

Runs pending migrations using the `app_migrate` role. Infisical injects the migration DB credentials:

```bash
ssh root@${SERVER_IP} "cd /opt/${PROJECT} && infisical run --env=prod -- docker compose exec app npm run migrate"
```

The `app_migrate` role has DDL privileges (CREATE, ALTER, DROP) but no data access, so migrations can't accidentally read/modify production data.

### up — Start/Restart Services

Adapt the start command to the secrets provider:

```bash
PROVIDER=$(jq -r '.secrets.provider' .deploy.json)

if [ "$PROVIDER" = "infisical" ]; then
  ssh root@${SERVER_IP} "cd /opt/${PROJECT} && infisical run --env=prod -- docker compose -f docker-compose.prod.yml up -d --build"
else
  # docker-secrets and s3-bundle inject secrets without infisical wrapper
  ssh root@${SERVER_IP} "cd /opt/${PROJECT} && docker compose -f docker-compose.prod.yml up -d --build"
fi
```

Regardless of provider, secrets never touch disk as plaintext and never appear in Docker image layers.

### status — Check Deployment Health

```bash
SERVER_IP=$(jq -r '.server.ip' .deploy.json)
PROJECT=$(jq -r '.project' .deploy.json)
HEALTH_URL=$(jq -r '.deploy.health_url' .deploy.json)

# Container status
ssh root@${SERVER_IP} "cd /opt/${PROJECT} && docker compose ps"

# Health endpoint
curl -sf "${HEALTH_URL}" && echo "  HEALTHY" || echo "  UNHEALTHY"

# Server resource usage
ssh root@${SERVER_IP} "free -h && echo '---' && df -h / && echo '---' && docker stats --no-stream"
```

### logs — View Service Logs

```bash
# Tail all services (last 100 lines, follow)
ssh root@${SERVER_IP} "cd /opt/${PROJECT} && docker compose logs -f --tail=100"

# Specific service
ssh root@${SERVER_IP} "cd /opt/${PROJECT} && docker compose logs -f --tail=100 {service_name}"
```

### rollback — Revert to Previous Version

Read `references/deploy-rollback.md` for the full playbook.

The server keeps the last 3 Docker image tags. To rollback:

```bash
# List available images
ssh root@${SERVER_IP} "docker images --format '{{.Repository}}:{{.Tag}} {{.CreatedAt}}' | grep ${PROJECT}"

# Rollback to previous tag
ssh root@${SERVER_IP} "cd /opt/${PROJECT} && docker compose -f docker-compose.prod.yml up -d --no-build"
```

Always run a health check after rollback.

### secrets — Manage Secrets

Read `references/deploy-secrets.md` for the full playbook. Commands vary by provider:

**Infisical:**
```bash
ssh root@${SERVER_IP} "infisical secrets list --env=prod"
ssh root@${SERVER_IP} "infisical secrets set KEY=value --env=prod"
```

**Docker Secrets:**
```bash
SECRETS_DIR=$(jq -r '.secrets.secrets_dir' .deploy.json)
ssh root@${SERVER_IP} "ls -la ${SECRETS_DIR}/"                    # list
ssh root@${SERVER_IP} "echo 'value' > ${SECRETS_DIR}/key_name"    # add/update
```

**S3 Bundle:**
```bash
# Download, decrypt, edit, re-encrypt, upload — see deploy-secrets.md for full flow
```

After changing secrets with any provider, restart services to pick them up.

## Deploy Flow

```
Read .deploy.json → rsync code → infisical injects secrets → docker compose up → Caddy auto-SSL → health check
```

## Safety Rules

- **Always read .deploy.json first** — never hardcode server IPs or project names
- **Health check after every deploy** — report pass/fail to the user
- **No secrets in rsync** — `.env` is excluded from sync. Secrets come from the configured provider.
- **Confirm before rollback** — show current vs previous version, ask user to confirm
- **No force-restart without checking** — always show `docker compose ps` before force-restarting
- **Never delete** containers/images without user approval
