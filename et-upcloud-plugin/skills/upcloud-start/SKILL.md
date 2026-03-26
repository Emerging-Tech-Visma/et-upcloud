---
name: upcloud-start
description: >
  Interactive onboarding wizard for UpCloud. Guides the user from zero to a fully provisioned
  and deployable project through a structured conversation: discovers requirements, recommends
  architecture, shows a plan for approval, provisions infrastructure, generates standalone
  management scripts, and provides an onboarding checklist. Use this skill when the user wants
  to start a new project on UpCloud, is new to the platform, says "set up everything from scratch",
  "I'm starting fresh", "walk me through setting up UpCloud", or wants a guided experience
  rather than jumping straight into provisioning.
---

# /upcloud:start — Interactive Onboarding Wizard

Guides a user from zero to a fully deployed project on UpCloud through a structured conversation. This wraps `/upcloud:setup` with a discovery phase, approval step, and script generation.

## Overview

```
Phase 1        Phase 2          Phase 3        Phase 4         Phase 5           Phase 6
DISCOVER  →  RECOMMEND  →  SHOW PLAN  →  PROVISION  →  GENERATE SCRIPTS  →  ONBOARD
questions     architecture     upctl          execute         deploy.sh          what's
about the     + cost           commands       the plan        migrate.sh         next
project       estimate         for review                     rollback.sh        checklist
```

---

## Phase 1 — Discovery

Ask the user these questions one group at a time. Don't dump all questions at once — have a conversation. Adapt follow-up questions based on answers.

### Group 1: The Project

Ask:
- **What are you building?** (web app, API, static site, full-stack app, internal tool)
- **What's the tech stack?** (Node/Bun, Python, Go, Rust, etc. + framework if known)
- **Is this a new project or are you migrating from somewhere?**

### Group 2: Data Needs

Based on the app type, ask relevant questions:
- **Do you need a database?** (most apps do — default yes)
  - If yes: **What kind of data?**
    - Relational/SQL → PostgreSQL (always)
    - Vector embeddings / AI / RAG → enable pgvector
    - Full-text or fuzzy search → enable pg_trgm
    - Scheduled jobs / cron → enable pg_cron
    - Flexible/document-like data → JSONB columns (built-in, no extension)
- **Do you need file uploads or object storage?** (images, documents, user content)

### Group 3: Scale & Environment

- **Expected traffic?** This determines the server and DB plan:
  - **Hobby / side project** → 1xCPU-2GB server, smallest DB plan
  - **Small team / internal tool** → 2xCPU-4GB server (default)
  - **Production / public-facing** → 4xCPU-8GB+ server, larger DB
  - **High traffic / scaling needed** → multiple servers + load balancer
- **Do you have a domain name ready?** (for HTTPS via Caddy)
  - If not: explain they can add it later, Caddy will use the server IP initially

### Group 4: Security & Compliance

- **Does this handle user data, payments, or have compliance needs?**
  - If yes → recommend Infisical (audit trail, rotation)
  - If no → recommend Docker Secrets (simpler, still low risk)
- **EU data residency required?** (default yes → `fi-hel1`)
  - If no: offer other zones

After each group, summarize what you've learned so far. This builds confidence that you understood correctly.

---

## Phase 2 — Architecture Recommendation

Based on discovery answers, build and present this table:

```
┌─────────────────────────────────────────────────────────────┐
│  Recommended Architecture for {project_name}                │
├──────────────────┬──────────────────────────────────────────┤
│ Server           │ {plan} in {zone}                         │
│ Database         │ PostgreSQL ({db_plan})                   │
│   Extensions     │ {list of extensions}                     │
│   Roles          │ app_rw, app_ro, app_migrate              │
│ Object Storage   │ {yes/no} — {reason}                     │
│ Secret Provider  │ {provider} — {reason}                   │
│ Load Balancer    │ {yes/no} — {reason}                     │
│ Domain           │ {domain or "configure later"}            │
│ Zone             │ {zone} ({location})                      │
├──────────────────┼──────────────────────────────────────────┤
│ Est. Monthly Cost│ ~€{total}/month                          │
│   Server         │ ~€{server_cost}                          │
│   Database       │ ~€{db_cost}                              │
│   Storage        │ ~€{storage_cost}                         │
└──────────────────┴──────────────────────────────────────────┘
```

**Ask the user:** "Does this look right? Want to change anything before we proceed?"

Adjust if they want changes. Repeat until they approve.

---

## Phase 3 — Provisioning Plan

Generate the exact commands that will run and show them:

```
I'll run these commands to set up your infrastructure:

1. upctl server create --hostname {project}-prod --zone {zone} --plan {plan} ...
2. upctl database create --title {project}-pg --type pg --plan {db_plan} ...
3. [if storage] upctl object-storage create --name {project}-uploads ...
4. SSH into server to install Docker + Caddy + {secret provider}
5. Create database roles (app_rw, app_ro, app_migrate)
6. Enable extensions: {extensions}
7. Store credentials in {secret provider}
8. Generate .deploy.json, docker-compose.prod.yml, Caddyfile
9. Generate management scripts in scripts/
```

**Ask the user:** "Ready to provision? This will create billable resources on your UpCloud account."

Wait for explicit approval before proceeding.

---

## Phase 4 — Provision

Delegate to the existing `/upcloud:setup` skill logic. Follow the provisioning playbooks in `../upcloud-setup/references/`:

1. `provision-server.md` — create server, install Docker + Caddy
2. `provision-db.md` — create managed PostgreSQL, enable extensions, create roles
3. `provision-storage.md` — create object storage (if requested)
4. `provision-secrets.md` — set up the chosen secret provider
5. Generate `.deploy.json` from template

Show progress as each step completes. If any step fails, report the error clearly and ask if the user wants to retry or skip.

---

## Phase 5 — Generate Scripts

Create standalone bash scripts in the project's `scripts/` directory. Read the templates from `templates/scripts/` and customize them with values from `.deploy.json`.

Scripts to generate:
- `scripts/deploy.sh` — rsync code + rebuild containers
- `scripts/migrate.sh` — run database migrations
- `scripts/rollback.sh` — revert to previous version
- `scripts/status.sh` — health check + container status
- `scripts/logs.sh` — tail container logs
- `scripts/secrets.sh` — manage secrets (list/add/update)

All scripts:
- Start with `#!/usr/bin/env bash` and `set -euo pipefail`
- Read `.deploy.json` for config (server IP, project name, provider)
- Are provider-aware (check `secrets.provider`)
- Have `--help` flags explaining usage
- Are executable (`chmod +x`)

After generating, make them executable:
```bash
chmod +x scripts/*.sh
```

---

## Phase 6 — Onboarding Checklist

Present a post-setup summary:

```
┌─────────────────────────────────────────────────────────────┐
│  {project_name} is ready!                                   │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Server:    ssh root@{server_ip}                            │
│  Database:  {db_host}:{db_port} (credentials in {provider})│
│  Storage:   {endpoint} (if created)                         │
│  Health:    {health_url}                                    │
│                                                             │
│  Next steps:                                                │
│  ┌───┐                                                      │
│  │ 1 │ Deploy your first version:                           │
│  └───┘   ./scripts/deploy.sh                                │
│          or: /upcloud:deploy push                           │
│                                                             │
│  ┌───┐                                                      │
│  │ 2 │ Add your secrets:                                    │
│  └───┘   ./scripts/secrets.sh add API_KEY=your_key          │
│          or: /upcloud:deploy secrets                        │
│                                                             │
│  ┌───┐                                                      │
│  │ 3 │ Point your domain:                                   │
│  └───┘   Add A record: {domain} → {server_ip}              │
│          Caddy will auto-provision SSL                      │
│                                                             │
│  ┌───┐                                                      │
│  │ 4 │ Check status anytime:                                │
│  └───┘   ./scripts/status.sh                                │
│          or: /upcloud:deploy status                         │
│                                                             │
│  Config saved to: .deploy.json (commit this to git)         │
│  Scripts saved to: scripts/ (commit these to git)           │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Safety

- **Never provision without explicit user approval** (Phase 3 gate)
- **Never delete resources** — this skill only creates
- **Show costs before creating** — no surprise bills
- **All credentials go to the secret provider** — never shown in plaintext after initial setup
