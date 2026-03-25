# Provision Secrets Playbook

Three supported secret management backends, ordered by risk (lowest first).

Read `.deploy.json` field `secrets.provider` to determine which backend is in use. If provisioning for the first time, the setup skill will have asked the user to choose.

---

## Option A: Infisical (self-hosted) — Lowest Risk

**Risk: Low** | Effort: Medium | Rotation: Yes | Audit trail: Yes

Infisical is deployed as a self-hosted Docker container on the UpCloud server. It manages all application secrets with AES-256-GCM encryption, audit trails, and rotation policies.

### A1. Deploy Infisical on the Server

```bash
ssh root@${SERVER_IP} "mkdir -p /opt/infisical"
scp templates/infisical-compose.yml root@${SERVER_IP}:/opt/infisical/docker-compose.yml
ssh root@${SERVER_IP} "cd /opt/infisical && docker compose up -d"
```

### A2. Initial Setup

Access Infisical at `https://{domain}/infisical` (via Caddy) or `http://{server_ip}:8080`.

1. Create admin account
2. Create organization
3. Create project matching the project name

### A3. Create Environments

Three environments per project:
- `dev` — local development
- `staging` — pre-production testing
- `prod` — production

### A4. Create Machine Identity

For automated deployments — never use personal tokens:

1. Organization Settings > Machine Identities
2. Create identity: `{project}-deploy`
3. Grant access to the project's `prod` environment
4. Save the client ID and secret

### A5. Store Bootstrap Token

On your Mac:
```bash
mkdir -p ~/.config/upcloud-deploy
echo "INFISICAL_TOKEN={machine_identity_token}" > ~/.config/upcloud-deploy/token
chmod 600 ~/.config/upcloud-deploy/token
```

On the server:
```bash
ssh root@${SERVER_IP} "mkdir -p /etc/infisical && echo 'INFISICAL_TOKEN={token}' > /etc/infisical/token && chmod 600 /etc/infisical/token"
```

### A6. Store Initial Secrets

```bash
ssh root@${SERVER_IP} <<'EOF'
infisical secrets set DATABASE_URL="postgresql://app_rw:{pass}@{host}:{port}/{db}?sslmode=require" --env=prod
infisical secrets set DATABASE_URL_RO="postgresql://app_ro:{pass}@{host}:{port}/{db}?sslmode=require" --env=prod
infisical secrets set DATABASE_URL_MIGRATE="postgresql://app_migrate:{pass}@{host}:{port}/{db}?sslmode=require" --env=prod
EOF
```

### A7. Configure Rotation

Via Infisical dashboard:
- Database passwords: 90-day rotation
- API keys: 30-day rotation
- Alert via webhook when rotation occurs

### A8. Deploy config in .deploy.json

```json
{
  "secrets": {
    "provider": "infisical",
    "project_id": "proj_abc123",
    "environment": "prod"
  }
}
```

### A9. How secrets reach containers

```bash
# Infisical injects env vars at process start — secrets never touch disk
infisical run --env=prod -- docker compose -f docker-compose.prod.yml up -d
```

### Security Model

- **At rest**: AES-256-GCM encryption in Infisical's own PostgreSQL database
- **In transit**: localhost only (same server). External access via Caddy TLS.
- **At runtime**: Injected into process environment via `infisical run --`. Never written to disk. Never in Docker image layers.
- **Access control**: Machine identities per project/environment. No shared tokens.
- **Audit**: Full trail of every secret access in Infisical dashboard.
- **Bootstrap**: The one manual secret is `INFISICAL_TOKEN` at `~/.config/upcloud-deploy/token` (chmod 600). This is the root of trust.

---

## Option B: Docker Secrets — Low Risk, Minimal Setup

**Risk: Low** | Effort: Low | Rotation: Manual | Audit trail: No

Docker Compose secrets are mounted as in-memory tmpfs files inside containers. Secrets never appear in image layers or `docker inspect` output.

### B1. Create Secret Files on Server

```bash
ssh root@${SERVER_IP} "mkdir -p /opt/${PROJECT}/secrets && chmod 700 /opt/${PROJECT}/secrets"

# Generate and store each secret
ssh root@${SERVER_IP} <<'EOF'
echo "postgresql://app_rw:{pass}@{host}:{port}/{db}?sslmode=require" > /opt/${PROJECT}/secrets/database_url
echo "postgresql://app_ro:{pass}@{host}:{port}/{db}?sslmode=require" > /opt/${PROJECT}/secrets/database_url_ro
echo "postgresql://app_migrate:{pass}@{host}:{port}/{db}?sslmode=require" > /opt/${PROJECT}/secrets/database_url_migrate
chmod 600 /opt/${PROJECT}/secrets/*
EOF
```

### B2. Configure docker-compose.prod.yml

```yaml
services:
  app:
    build: .
    secrets:
      - database_url
      - database_url_ro
    environment:
      - NODE_ENV=production
      - PORT=8080

secrets:
  database_url:
    file: ./secrets/database_url
  database_url_ro:
    file: ./secrets/database_url_ro
  database_url_migrate:
    file: ./secrets/database_url_migrate
```

### B3. Read Secrets in Application

Secrets are mounted at `/run/secrets/{name}` inside the container:

```typescript
import { readFileSync } from 'fs';

const DATABASE_URL = readFileSync('/run/secrets/database_url', 'utf8').trim();
```

Or use a helper that checks both env var and secret file:
```typescript
function getSecret(name: string): string {
  // Check env var first (for local dev)
  if (process.env[name.toUpperCase()]) return process.env[name.toUpperCase()]!;
  // Check Docker secret file
  try { return readFileSync(`/run/secrets/${name}`, 'utf8').trim(); }
  catch { throw new Error(`Secret ${name} not found`); }
}
```

### B4. Deploy config in .deploy.json

```json
{
  "secrets": {
    "provider": "docker-secrets",
    "secrets_dir": "/opt/{project}/secrets"
  }
}
```

### B5. How secrets reach containers

```bash
# Docker mounts secrets as tmpfs — they exist only in memory
docker compose -f docker-compose.prod.yml up -d --build
```

### Limitations

- No automatic rotation — you must manually update secret files and restart
- No audit trail — no record of who accessed which secret when
- No centralized management — secrets live as files on the server
- To rotate: update file → `docker compose restart`

---

## Option C: UpCloud Object Storage Bundle — Acceptable Risk

**Risk: Acceptable** | Effort: Low | Rotation: Manual (you build it) | Audit trail: No

Encrypted secret bundle stored in S3-compatible Object Storage. Fetched at container start via an entrypoint script.

### C1. Create and Encrypt Bundle

```bash
# Create secrets file locally (temporary)
cat > /tmp/secrets.env <<'EOF'
DATABASE_URL=postgresql://app_rw:{pass}@{host}:{port}/{db}?sslmode=require
DATABASE_URL_RO=postgresql://app_ro:{pass}@{host}:{port}/{db}?sslmode=require
S3_ACCESS_KEY={key}
S3_SECRET_KEY={secret}
EOF

# Encrypt with age (or gpg)
age -p -o /tmp/secrets.env.age /tmp/secrets.env

# Upload to Object Storage
aws s3 cp /tmp/secrets.env.age s3://{project}-secrets/prod/secrets.env.age \
  --endpoint-url https://{project}-uploads.{zone}.upcloudobjects.com

# Clean up local plaintext immediately
rm /tmp/secrets.env
```

### C2. Entrypoint Script

Add to your Docker image:

```bash
#!/bin/sh
# fetch-secrets.sh — runs before app starts
aws s3 cp s3://${SECRETS_BUCKET}/prod/secrets.env.age /tmp/secrets.env.age \
  --endpoint-url ${S3_ENDPOINT}
age -d -i /run/secrets/age_key /tmp/secrets.env.age > /tmp/secrets.env
set -a && source /tmp/secrets.env && set +a
rm /tmp/secrets.env /tmp/secrets.env.age
exec "$@"
```

### C3. Deploy config in .deploy.json

```json
{
  "secrets": {
    "provider": "s3-bundle",
    "bucket": "{project}-secrets",
    "endpoint": "https://{project}-uploads.{zone}.upcloudobjects.com",
    "encryption": "age"
  }
}
```

### Limitations

- You own the encryption and rotation logic
- No built-in audit trail (though S3 access logs can help)
- Requires the decryption key to be available on the server
- More moving parts than Docker Secrets

---

## Comparison Summary

| Aspect | Infisical | Docker Secrets | S3 Bundle |
| ------ | --------- | -------------- | --------- |
| Secrets on disk | Never | tmpfs only | Encrypted at rest |
| Secrets in image | Never | Never | Never |
| Rotation | Automatic | Manual | Manual (you build) |
| Audit trail | Full | None | S3 access logs |
| Runtime injection | `infisical run --` | `/run/secrets/` mount | Entrypoint script |
| Dependencies | Infisical container | Docker (already there) | S3 SDK + age/gpg |
| Compliance-ready | Yes | Partial | Partial |

**Default recommendation: Infisical** for production workloads. **Docker Secrets** for simple/internal apps where audit trails aren't required.

❗ **Never use encrypted .env baked into containers** — secrets end up in image layers, no rotation, no audit trail, anyone with image access has all credentials.
