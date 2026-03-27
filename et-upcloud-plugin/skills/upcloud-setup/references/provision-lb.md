# Provision Load Balancer Playbook

Only create a load balancer when the project needs horizontal scaling (multiple app servers).

## Step 1: Check Available Plans

```bash
upctl load-balancer plans
```

## Step 2: Create Load Balancer

Load balancer creation is managed through the UpCloud control panel or API, as the CLI has limited LB create support. Use `upctl` for listing and management:

```bash
# List existing load balancers
upctl load-balancer list

# Show details
upctl load-balancer show {uuid}
```

## Step 3: Configure via Control Panel

1. Go to UpCloud Control Panel > Load Balancers
2. Create new LB in the same zone as your servers
3. Configure:
   - **Frontend**: HTTPS on port 443, HTTP on port 80 (redirect to HTTPS)
   - **Backend**: Your app servers on the Docker port (typically 8080)
   - **Health check**: GET /health, interval 10s, threshold 3
   - **TLS**: Upload or use Let's Encrypt certificate

## When to Use

- Single server: Skip LB, use Caddy directly for TLS termination
- Multiple servers: Use LB for distribution + Caddy on each for local routing
- The `/upcloud:setup` skill only provisions an LB if the user explicitly requests scaling support

## Notes

- LB pricing varies by plan — check `upctl load-balancer plans`
- For most single-server projects, Caddy alone handles TLS and reverse proxy perfectly
