# Deploy Secrets Playbook

Manage application secrets on the server. Commands vary by provider — check `secrets.provider` in `.deploy.json`.

## Detect Provider

```bash
SERVER_IP=$(jq -r '.server.ip' .deploy.json)
PROJECT=$(jq -r '.project' .deploy.json)
PROVIDER=$(jq -r '.secrets.provider' .deploy.json)
```

---

## Infisical Provider

### List Secrets
```bash
ssh root@${SERVER_IP} "infisical secrets list --env=prod"
ssh root@${SERVER_IP} "infisical secrets list --env=staging"
```

### Add / Update
```bash
ssh root@${SERVER_IP} "infisical secrets set KEY=value --env=prod"
```

### Delete
**Always confirm with the user first.**
```bash
ssh root@${SERVER_IP} "infisical secrets delete SECRET_NAME --env=prod"
```

### Restart to Apply
```bash
ssh root@${SERVER_IP} "cd /opt/${PROJECT} && infisical run --env=prod -- docker compose -f docker-compose.prod.yml up -d"
```

---

## Docker Secrets Provider

### List Secrets
```bash
SECRETS_DIR=$(jq -r '.secrets.secrets_dir' .deploy.json)
ssh root@${SERVER_IP} "ls -la ${SECRETS_DIR}/"
```

### Add / Update
```bash
ssh root@${SERVER_IP} "echo 'secret_value' > ${SECRETS_DIR}/key_name && chmod 600 ${SECRETS_DIR}/key_name"
```

### Delete
**Always confirm with the user first.**
```bash
ssh root@${SERVER_IP} "rm ${SECRETS_DIR}/key_name"
```

### Restart to Apply
Docker Secrets are mounted at container start, so a restart picks up changes:
```bash
ssh root@${SERVER_IP} "cd /opt/${PROJECT} && docker compose -f docker-compose.prod.yml up -d --force-recreate"
```

### Reading in App Code
Secrets are at `/run/secrets/{name}` inside the container:
```typescript
import { readFileSync } from 'fs';
const DB_URL = readFileSync('/run/secrets/database_url', 'utf8').trim();
```

---

## S3 Bundle Provider

### List Secrets
Download and decrypt the bundle to view:
```bash
ssh root@${SERVER_IP} <<'EOF'
BUCKET=$(jq -r '.secrets.bucket' /opt/${PROJECT}/.deploy.json)
ENDPOINT=$(jq -r '.secrets.endpoint' /opt/${PROJECT}/.deploy.json)
aws s3 cp s3://${BUCKET}/prod/secrets.env.age /tmp/secrets.env.age --endpoint-url ${ENDPOINT}
age -d -i /etc/age/key.txt /tmp/secrets.env.age
rm /tmp/secrets.env.age
EOF
```

### Add / Update / Delete
1. Download and decrypt the bundle
2. Edit the plaintext file
3. Re-encrypt and upload
4. Restart containers

```bash
# Full cycle on server
ssh root@${SERVER_IP} <<'EOF'
BUCKET=$(jq -r '.secrets.bucket' /opt/${PROJECT}/.deploy.json)
ENDPOINT=$(jq -r '.secrets.endpoint' /opt/${PROJECT}/.deploy.json)
aws s3 cp s3://${BUCKET}/prod/secrets.env.age /tmp/secrets.env.age --endpoint-url ${ENDPOINT}
age -d -i /etc/age/key.txt /tmp/secrets.env.age > /tmp/secrets.env
# Edit /tmp/secrets.env as needed
age -r $(cat /etc/age/key.txt.pub) -o /tmp/secrets.env.age /tmp/secrets.env
aws s3 cp /tmp/secrets.env.age s3://${BUCKET}/prod/secrets.env.age --endpoint-url ${ENDPOINT}
rm /tmp/secrets.env /tmp/secrets.env.age
EOF
```

### Restart to Apply
```bash
ssh root@${SERVER_IP} "cd /opt/${PROJECT} && docker compose -f docker-compose.prod.yml up -d --force-recreate"
```

---

## Common Secrets

Typical secrets stored for a project (regardless of provider):

| Key | Description |
| --- | --- |
| `DATABASE_URL` | App read-write connection string |
| `DATABASE_URL_RO` | Read-only connection string |
| `DATABASE_URL_MIGRATE` | Migration role connection string |
| `S3_ACCESS_KEY` | Object storage access key |
| `S3_SECRET_KEY` | Object storage secret key |
| `SESSION_SECRET` | Application session signing key |
| `API_KEY` | Third-party API keys |
| `SMTP_PASSWORD` | Email service credentials |

## Security Rules (All Providers)

- **Never echo secrets** to stdout or logs
- **Never store secrets in .env files** on the server or in Docker images
- **Never use encrypted .env baked into containers** — this is the high-risk option we explicitly avoid
- **Rotate regularly** — 90 days for DB passwords, 30 days for API keys
- **Use scoped credentials** — machine identities (Infisical), per-service files (Docker), scoped S3 keys (bundle)
