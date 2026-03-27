# Provision Database Playbook

## Step 1: Create Managed PostgreSQL

```bash
upctl database create \
  --title "{project}-pg" \
  --zone "{zone}" \
  --hostname-prefix "{project}-pg" \
  --type pg \
  --plan "2x2xCPU-4GB-100GB" \
  --maintenance-dow monday \
  --maintenance-time 03:00:00 \
  --enable-termination-protection \
  --wait
```

## Step 2: Get Connection Details

```bash
DB_INFO=$(upctl database show "{project}-pg" -o json)
DB_HOST=$(echo "$DB_INFO" | jq -r '.components[] | select(.component=="pg") | .host')
DB_PORT=$(echo "$DB_INFO" | jq -r '.components[] | select(.component=="pg") | .port')
DB_USER=$(echo "$DB_INFO" | jq -r '.users[0].username')
DB_PASS=$(echo "$DB_INFO" | jq -r '.users[0].password')
echo "Host: ${DB_HOST}:${DB_PORT}"
```

## Step 3: Enable Extensions

Connect to the database and enable extensions:

```sql
-- Vector store for embeddings and similarity search
CREATE EXTENSION IF NOT EXISTS vector;

-- Scheduled jobs in SQL
CREATE EXTENSION IF NOT EXISTS pg_cron;

-- Fuzzy text search (trigram matching)
CREATE EXTENSION IF NOT EXISTS pg_trgm;

-- UUID generation
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
```

## Step 4: Create Application Database

```sql
CREATE DATABASE {project};
```

## Step 5: Create Roles

```sql
-- Read-write role for the application
CREATE ROLE app_rw LOGIN PASSWORD '{generated_password}';
GRANT CONNECT ON DATABASE {project} TO app_rw;
GRANT USAGE ON SCHEMA public TO app_rw;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO app_rw;
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT SELECT, INSERT, UPDATE, DELETE ON TABLES TO app_rw;
GRANT USAGE, SELECT ON ALL SEQUENCES IN SCHEMA public TO app_rw;
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT USAGE, SELECT ON SEQUENCES TO app_rw;

-- Read-only role for reporting/analytics
CREATE ROLE app_ro LOGIN PASSWORD '{generated_password}';
GRANT CONNECT ON DATABASE {project} TO app_ro;
GRANT USAGE ON SCHEMA public TO app_ro;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO app_ro;
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT SELECT ON TABLES TO app_ro;

-- Migration role for schema changes only
CREATE ROLE app_migrate LOGIN PASSWORD '{generated_password}';
GRANT CONNECT ON DATABASE {project} TO app_migrate;
GRANT USAGE, CREATE ON SCHEMA public TO app_migrate;
GRANT ALL ON ALL TABLES IN SCHEMA public TO app_migrate;
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT ALL ON TABLES TO app_migrate;
GRANT ALL ON ALL SEQUENCES IN SCHEMA public TO app_migrate;
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT ALL ON SEQUENCES TO app_migrate;
```

## Step 6: Store Credentials in Infisical

Store each role's connection string in Infisical:
```
DATABASE_URL=postgresql://app_rw:{password}@{host}:{port}/{project}?sslmode=require
DATABASE_URL_RO=postgresql://app_ro:{password}@{host}:{port}/{project}?sslmode=require
DATABASE_URL_MIGRATE=postgresql://app_migrate:{password}@{host}:{port}/{project}?sslmode=require
```

## Notes

- Managed PostgreSQL includes automatic daily backups with 7-day retention
- The `--enable-termination-protection` flag prevents accidental deletion
- UpCloud managed databases support SSL by default — always use `sslmode=require`
- Available plans: `upctl database plans --type pg`
- Role passwords should be generated randomly (use `openssl rand -base64 32`)
