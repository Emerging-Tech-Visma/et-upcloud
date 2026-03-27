---
name: deploy
description: Deploy current project to UpCloud
arguments: command
---

# UpCloud Deploy

Run deploy command: `$ARGUMENTS`

## Commands

- `push` — Sync code and restart services
- `migrate` — Run database migrations
- `up` — Start/restart Docker services
- `status` — Check deployment health
- `logs` — View service logs
- `rollback` — Revert to previous version
- `secrets` — Manage Infisical secrets

## Steps

1. Read `.deploy.json` to get server details
2. Read the `/upcloud:deploy` skill at `.claude/skills/upcloud-deploy/SKILL.md`
3. Execute the requested command
4. Report results

## Examples

```
/deploy push
/deploy status
/deploy logs
/deploy rollback
/deploy secrets
```
