---
name: server-status
description: Quick server status check
---

# Server Status

Check all UpCloud resources for the current project.

## Steps

1. Read `.deploy.json`
2. Run these checks:

```bash
# Server status
upctl server show $(jq -r '.server.hostname' .deploy.json) -o json | jq '{state, hostname, plan, zone}'

# Database status
upctl database show $(jq -r '.database.uuid' .deploy.json) -o json | jq '{state, title, plan, zone}'

# Container status
ssh root@$(jq -r '.server.ip' .deploy.json) "docker ps --format 'table {{.Names}}\t{{.Status}}\t{{.Ports}}'"

# Health check
curl -sf $(jq -r '.deploy.health_url' .deploy.json) && echo "HEALTHY" || echo "UNHEALTHY"
```

3. Report results in a summary table
