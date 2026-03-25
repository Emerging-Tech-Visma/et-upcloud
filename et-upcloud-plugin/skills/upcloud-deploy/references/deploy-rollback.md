# Deploy Rollback Playbook

## Rollback Strategy

The server keeps the last 3 Docker image builds. Rollback restarts containers with the previous image without rebuilding.

### 1. Show Current State

```bash
SERVER_IP=$(jq -r '.server.ip' .deploy.json)
PROJECT=$(jq -r '.project' .deploy.json)

# Current running containers
ssh root@${SERVER_IP} "cd /opt/${PROJECT} && docker compose ps"

# Available image versions (most recent first)
ssh root@${SERVER_IP} "docker images --format 'table {{.Repository}}\t{{.Tag}}\t{{.CreatedAt}}\t{{.Size}}' | grep ${PROJECT}"
```

### 2. Confirm with User

Show the user:
- Current running version (image ID + created date)
- Previous available versions
- Ask: "Roll back to [version]?"

**Never rollback without user confirmation.**

### 3. Execute Rollback

```bash
# Stop current containers
ssh root@${SERVER_IP} "cd /opt/${PROJECT} && docker compose down"

# Start with previous image (no --build flag = uses existing image)
ssh root@${SERVER_IP} "cd /opt/${PROJECT} && infisical run --env=prod -- docker compose -f docker-compose.prod.yml up -d"
```

If you need a specific older image:
```bash
# Tag the older image as latest
ssh root@${SERVER_IP} "docker tag ${PROJECT}-app:{old_tag} ${PROJECT}-app:latest"

# Restart
ssh root@${SERVER_IP} "cd /opt/${PROJECT} && infisical run --env=prod -- docker compose -f docker-compose.prod.yml up -d"
```

### 4. Verify

```bash
# Check containers are running
ssh root@${SERVER_IP} "cd /opt/${PROJECT} && docker compose ps"

# Health check
HEALTH_URL=$(jq -r '.deploy.health_url' .deploy.json)
curl -sf "${HEALTH_URL}" && echo "HEALTHY" || echo "UNHEALTHY"
```

### 5. Cleanup Old Images

Keep last 3 images, prune the rest:
```bash
ssh root@${SERVER_IP} "docker image prune -f --filter 'until=168h'"
```

## Database Rollback

If the deployment included a migration that needs reverting:

1. Check if the migration tool supports `down` migrations
2. Run the down migration with the `app_migrate` role
3. This is framework-specific and should be confirmed with the user

**Database rollbacks are dangerous** — always get explicit approval and consider whether the migration was destructive (data loss).
