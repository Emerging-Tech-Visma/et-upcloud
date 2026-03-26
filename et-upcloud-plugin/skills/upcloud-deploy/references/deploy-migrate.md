# Deploy Migration Playbook

## Running Migrations

Migrations use the `app_migrate` role which has DDL privileges (CREATE, ALTER, DROP tables) but is separate from the application's runtime role.

### 1. Read Config

```bash
SERVER_IP=$(jq -r '.server.ip' .deploy.json)
PROJECT=$(jq -r '.project' .deploy.json)
```

### 2. Run Migrations

The migration command depends on your framework. Infisical injects `DATABASE_URL_MIGRATE` at runtime:

**Generic (via Docker exec):**
```bash
ssh root@${SERVER_IP} "cd /opt/${PROJECT} && infisical run --env=prod -- docker compose exec app sh -c 'DATABASE_URL=\$DATABASE_URL_MIGRATE npm run migrate'"
```

**Prisma:**
```bash
ssh root@${SERVER_IP} "cd /opt/${PROJECT} && infisical run --env=prod -- docker compose exec app sh -c 'DATABASE_URL=\$DATABASE_URL_MIGRATE npx prisma migrate deploy'"
```

**Drizzle:**
```bash
ssh root@${SERVER_IP} "cd /opt/${PROJECT} && infisical run --env=prod -- docker compose exec app sh -c 'DATABASE_URL=\$DATABASE_URL_MIGRATE npx drizzle-kit push'"
```

**Raw SQL files:**
```bash
ssh root@${SERVER_IP} "cd /opt/${PROJECT} && infisical run --env=prod -- docker compose exec app sh -c 'for f in migrations/*.sql; do psql \$DATABASE_URL_MIGRATE -f \$f; done'"
```

### 3. Verify

```bash
# Check migration status (Prisma example)
ssh root@${SERVER_IP} "cd /opt/${PROJECT} && infisical run --env=prod -- docker compose exec app sh -c 'DATABASE_URL=\$DATABASE_URL_MIGRATE npx prisma migrate status'"
```

## Safety

- Always run migrations **before** deploying new app code that depends on schema changes
- The `app_migrate` role has DDL access but the `app_rw` role does not — this prevents the running application from accidentally modifying the schema
- For destructive migrations (DROP TABLE, DROP COLUMN), always ask the user for confirmation
- Consider running a dry-run or generating the SQL first for review
