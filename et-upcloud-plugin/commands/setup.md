---
name: setup
description: Provision UpCloud infrastructure for a project
arguments: project_name
---

# UpCloud Setup

Provision infrastructure for project `$ARGUMENTS`.

## Steps

1. Read the `/upcloud:setup` skill at `.claude/skills/upcloud-setup/SKILL.md`
2. Follow the skill's workflow to provision:
   - Cloud Server (Docker + Caddy + Infisical)
   - Managed PostgreSQL (with extensions)
   - Object Storage (if needed)
   - Infisical secrets management
3. Generate `.deploy.json` config

## Example

```
/setup workbench
```
