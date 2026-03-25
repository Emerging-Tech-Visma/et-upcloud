# Deploy Push Playbook

## Full Push Sequence

### 1. Read Config

```bash
SERVER_IP=$(jq -r '.server.ip' .deploy.json)
PROJECT=$(jq -r '.project' .deploy.json)
HEALTH_URL=$(jq -r '.deploy.health_url' .deploy.json)
COMPOSE_FILE=$(jq -r '.deploy.compose_file // "docker-compose.prod.yml"' .deploy.json)
```

### 2. Pre-flight Checks

```bash
# Verify server is reachable
ssh -o ConnectTimeout=5 root@${SERVER_IP} 'echo ok' || { echo "Server unreachable"; exit 1; }

# Verify docker is running
ssh root@${SERVER_IP} 'docker info > /dev/null 2>&1' || { echo "Docker not running"; exit 1; }
```

### 3. Sync Code

```bash
rsync -avz --delete \
  --exclude '.git' \
  --exclude 'node_modules' \
  --exclude '.env' \
  --exclude '.env.*' \
  --exclude '.deploy.json' \
  --exclude '*.log' \
  --exclude '.DS_Store' \
  -e "ssh -o StrictHostKeyChecking=accept-new" \
  ./ root@${SERVER_IP}:/opt/${PROJECT}/
```

The `--delete` flag removes files on the server that no longer exist locally, keeping the deployment clean.

### 4. Build and Restart

```bash
ssh root@${SERVER_IP} "cd /opt/${PROJECT} && infisical run --env=prod -- docker compose -f ${COMPOSE_FILE} up -d --build"
```

If Infisical is not configured yet (no secrets needed), fall back to:
```bash
ssh root@${SERVER_IP} "cd /opt/${PROJECT} && docker compose -f ${COMPOSE_FILE} up -d --build"
```

### 5. Health Check

```bash
# Wait for services to start
sleep 5

# Check container status
ssh root@${SERVER_IP} "cd /opt/${PROJECT} && docker compose ps --format json" | jq '.'

# Check health endpoint
if [ "${HEALTH_URL}" != "null" ]; then
  for i in 1 2 3; do
    if curl -sf "${HEALTH_URL}" > /dev/null; then
      echo "Health check passed"
      break
    fi
    sleep 3
  done
fi
```

### 6. Show Summary

Report to the user:
- Which files were synced (rsync output)
- Container status (running/restarting/exited)
- Health check result
- Deploy timestamp

## Rsync Exclude Patterns

Default excludes (always applied):
- `.git` — version control history
- `node_modules` — rebuilt on server
- `.env`, `.env.*` — secrets must come from Infisical
- `.deploy.json` — local config, not needed on server
- `*.log` — local log files
- `.DS_Store` — macOS metadata

Custom excludes can be added via `.deploy.json`:
```json
{
  "deploy": {
    "rsync_exclude": ["data/", "tmp/", "coverage/"]
  }
}
```
