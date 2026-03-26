---
name: start
description: Interactive onboarding wizard — from zero to deployed on UpCloud
---

# UpCloud Onboarding Wizard

Start a guided setup for a new project on UpCloud.

## Steps

1. Read the `/upcloud:start` skill at `skills/upcloud-start/SKILL.md`
2. Follow the 6-phase wizard:
   - **Discover** — ask about the project, stack, data needs, scale, compliance
   - **Recommend** — show architecture + cost estimate
   - **Plan** — show exact commands for user approval
   - **Provision** — create the infrastructure
   - **Scripts** — generate deploy.sh, migrate.sh, rollback.sh, etc.
   - **Onboard** — show "what's next" checklist

## Example

```
/start
```

For users who already know what they want, use `/setup` instead.
