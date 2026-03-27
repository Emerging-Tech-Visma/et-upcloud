# Changelog

## v1.0.0 (2026-03-27)

Initial release.

- `/upcloud:start` — interactive onboarding wizard (discover → recommend → plan → provision → scripts → onboard)
- `/upcloud:setup` — provision servers, managed PostgreSQL, object storage, secrets, DB roles
- `/upcloud:deploy` — push, migrate, up, status, logs, rollback, secrets
- Standalone bash scripts: deploy.sh, migrate.sh, rollback.sh, status.sh, logs.sh, secrets.sh
- Scripts are provider-aware (Infisical, Docker Secrets, S3 Bundle) and read .deploy.json
- Three low-risk secret providers: Infisical, Docker Secrets, S3 Bundle (encrypted .env never supported)
- Full `upctl` CLI reference (verified against official UpCloud CLI)
- Supported databases: PostgreSQL (+ pgvector, pg_cron, pg_trgm), MySQL, OpenSearch, Valkey (Redis not supported — deprecated upstream)
- EU-only object storage regions (FI-HEL2, SE-STO1, DE-FRA1)
- Templates: docker-compose.prod.yml, Caddyfile, infisical-compose.yml, .deploy.json
- Deletion policy: all `upctl delete` commands require explicit user approval (deny list in settings.json)
- Pre-flight validation: `claude plugin validate` required before PR or deploy
