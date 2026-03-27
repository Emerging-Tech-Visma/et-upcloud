---
name: upcloud-setup
description: >
  Provision full-stack infrastructure on UpCloud using the upctl CLI. Creates cloud servers,
  managed PostgreSQL databases (with pgvector, pg_cron, pg_trgm), S3-compatible object storage,
  Infisical secret management, database roles, and optional load balancers. Generates a .deploy.json
  config file for the project. Use this skill whenever the user wants to set up infrastructure on
  UpCloud, provision a new server, create a database, set up object storage, configure secrets
  management, or bootstrap a new deployment project — even if they just say "set up the backend"
  or "create the infrastructure" or "provision a server".
---

# /upcloud:setup — Infrastructure Provisioning

Provisions the full backend stack on UpCloud for a project. Asks the user which features the project needs, then executes the provisioning sequence using `upctl`.

## Prerequisites

- `upctl` installed and authenticated (`upctl account login --with-token` or `UPCLOUD_TOKEN` env var)
- SSH key available (`~/.ssh/id_ed25519.pub` or similar)
- For Infisical: bootstrap token at `~/.config/upcloud-deploy/token`

Verify prerequisites before starting:
```bash
upctl account show -o json | jq '.username'
```

## Workflow

### 1. Gather Requirements

Ask the user:
- **Project name** — used for hostnames, DB names, bucket names (lowercase, alphanumeric + hyphens)
- **Zone** — default `fi-hel1` (Helsinki). Options: `fi-hel1`, `de-fra1`, `nl-ams1`, `uk-lon1`, `us-nyc1`, `us-chi1`, `sg-sin1`
- **Server plan** — default `2xCPU-4GB`. List with: `upctl server plans`
- **Features needed**:
  - PostgreSQL database (default: yes)
  - Object storage (default: no)
  - Load balancer (default: no)
- **Secret management strategy** — present the options table below, recommend based on project needs
- **Domain name** — for Caddy auto-SSL (can be added later)

#### Secret Management Decision

Present this table and help the user choose. Default to the lowest-risk option that fits their effort budget:

| Option | How it works | Effort | Risk | Best for |
| ------ | ------------ | ------ | ---- | -------- |
| **Infisical (self-hosted)** | Container on same server, inject via `infisical run` | Medium | **Low** — rotation, audit trail, open source | Production apps with compliance needs |
| **Docker Secrets** | Native Compose secrets, mounted as tmpfs at runtime | Low (already using Compose) | **Low** — no third-party dep, but no rotation or audit | Simple apps, single-server setups |
| **UpCloud Object Storage** | Encrypted secret bundle in S3, fetch at runtime via scoped keys | Low | **Acceptable** — you own encryption/rotation logic | Teams with existing S3 tooling |
**Recommendation logic:**
- If the project handles user data, payments, or has compliance needs → **Infisical** (lowest risk, full audit trail)
- If the project is simple/internal and the user wants minimal setup → **Docker Secrets** (low risk, zero dependencies)
- If the user already has S3 encryption tooling → **UpCloud Object Storage** (acceptable risk)

❗ **Encrypted .env in containers is never an option** — secrets end up in image layers, no rotation, no audit trail. This skill does not support it.

The chosen strategy is stored in `.deploy.json` under `secrets.provider` (`infisical`, `docker-secrets`, or `s3-bundle`).

### 2. Check for Existing Config

Read `.deploy.json` in the project root. If it exists, skip resources that are already provisioned (idempotent). Show the user what already exists and what will be created.

### 3. Provision Resources

Execute in order. Each step checks if the resource already exists before creating.

#### provision-server

Read `references/provision-server.md` for the full playbook.

**Quick reference:**
```bash
upctl server create \
  --hostname "{project}-prod" \
  --zone "{zone}" \
  --plan "{plan}" \
  --os "Ubuntu Server 24.04 LTS (Noble Numbat)" \
  --ssh-keys ~/.ssh/id_ed25519.pub \
  --enable-firewall \
  --enable-metadata \
  --wait
```

After creation, SSH in to install Docker + Caddy + Infisical CLI.

#### provision-db

Read `references/provision-db.md` for the full playbook.

**Quick reference:**
```bash
upctl database create \
  --title "{project}-pg" \
  --zone "{zone}" \
  --hostname-prefix "{project}-pg" \
  --type pg \
  --plan "2x2xCPU-4GB-100GB" \
  --wait
```

Then connect and enable extensions (`vector`, `pg_cron`, `pg_trgm`) and create roles.

#### provision-storage (if requested)

Read `references/provision-storage.md` for the full playbook.

**Important:** Object storage regions differ from server zones. Map the server zone to the nearest EU region:

| Server Zone | Object Storage Region |
|-------------|----------------------|
| `fi-hel1`   | `FI-HEL2`           |
| `se-sto1`   | `SE-STO1`           |
| `de-fra1`   | `DE-FRA1`           |

Only EU regions are allowed (EU data residency policy). Run `upctl object-storage regions` to verify availability.

**Quick reference:**
```bash
upctl object-storage create \
  --name "{project}-uploads" \
  --region "{region}" \
  --network type=public,name=public,family=IPv4 \
  --wait
```

Then create user, access keys, and default bucket.

#### provision-secrets

Read `references/provision-secrets.md` for the full playbook. The playbook covers all three supported backends:

- **Infisical**: Deploy container, create project + environments (dev/staging/prod), store initial secrets, configure machine identity
- **Docker Secrets**: Create secrets in Docker Compose, configure tmpfs mounts, generate secret files
- **S3 Bundle**: Create encrypted bundle, upload to Object Storage, configure fetch-at-runtime script

#### provision-roles

Create three PostgreSQL roles:
- `app_rw` — read-write for the app
- `app_ro` — read-only for analytics
- `app_migrate` — DDL only (CREATE, ALTER, DROP)

#### provision-lb (optional)

Only if user requested scaling support:
```bash
upctl load-balancer plans   # show available plans first
```

### 4. Generate Config

Write `.deploy.json` to the project root. Read `templates/.deploy.json.template` for the schema.

Also generate from templates:
- `docker-compose.prod.yml`
- `Caddyfile`

### 5. Verify

```bash
# Server reachable
ssh -o ConnectTimeout=5 root@{server_ip} 'echo ok'

# Database running
upctl database show "{project}-pg" -o json | jq '.state'

# Storage ready (if created)
upctl object-storage show "{project}-uploads" -o json | jq '.configured_status'
```

Report results to the user with a summary table.

## Cost Estimates

Show before provisioning:

| Resource              | Plan              | ~EUR/month |
| --------------------- | ----------------- | ---------- |
| Cloud Server          | 2xCPU-4GB         | ~22        |
| Managed PostgreSQL    | 1xCPU-2GB-25GB    | ~16        |
| Object Storage        | 250GB             | ~5         |
| **Total (typical)**   |                   | **~43**    |

Multiple lightweight projects can share one server + one PG instance (separate databases).

## Safety Rules

- **Never delete existing resources** — only create or skip
- **Always confirm** before creating resources that cost money
- **Store no secrets in .deploy.json** — only UUIDs, hostnames, endpoints
- **Default to EU zones** — `fi-hel1` unless user specifies otherwise
- **Require user approval** for any `upctl ... delete` command
