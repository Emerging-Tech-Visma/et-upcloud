# Changelog

## v1.2.0 (2026-03-25)

- Restructured as Claude Code marketplace plugin
- Added README with installation instructions
- Added CHANGELOG
- Set up branch protection (PR-only merges to main)

## v1.0.0 (2026-03-25)

- Initial release
- `/upcloud:setup` skill — provision servers, managed PostgreSQL, object storage, secrets, DB roles
- `/upcloud:deploy` skill — push, migrate, up, status, logs, rollback, secrets
- Three low-risk secret providers: Infisical, Docker Secrets, S3 Bundle
- Full `upctl` CLI reference
- Templates: docker-compose.prod.yml, Caddyfile, infisical-compose.yml, .deploy.json
- PostgreSQL extensions guide (pgvector, pg_cron, pg_trgm, JSONB)
