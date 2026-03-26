# Changelog

## v1.0.0 (2026-03-26)

Initial public release.

- `/upcloud:setup` skill — provision servers, managed PostgreSQL, object storage, secrets, DB roles
- `/upcloud:deploy` skill — push, migrate, up, status, logs, rollback, secrets
- Three low-risk secret providers: Infisical, Docker Secrets, S3 Bundle (encrypted .env never supported)
- Full `upctl` CLI reference
- Templates: docker-compose.prod.yml, Caddyfile, infisical-compose.yml, .deploy.json
- PostgreSQL extensions guide (pgvector, pg_cron, pg_trgm, JSONB)
- Structured as Claude Code marketplace plugin
- Deletion policy: all `upctl delete` commands require explicit user approval (deny list in settings.json)
- Branch protection: main requires PR (no direct pushes)
- Pre-flight validation: `claude plugin validate` required before PR or deploy
