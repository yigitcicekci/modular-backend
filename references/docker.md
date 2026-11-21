# Docker and local development

Use a production `Dockerfile` with a build stage and a non-root Node runtime stage. Reuse the [release image example](ci-cd.md#integration-tests-and-containers) and the repository's intended versions. For local development, keep Compose and Bash scripts in the repo. This example runs the Nest watch process on the host and PostgreSQL and Redis in Compose. Keep an equivalent existing workflow during revisions.

## Compose dependencies

```yaml
# docker/compose.yaml
services:
  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: ${POSTGRES_USER:?Set POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:?Set POSTGRES_PASSWORD}
      POSTGRES_DB: ${POSTGRES_DB:?Set POSTGRES_DB}
    ports:
      - "127.0.0.1:${POSTGRES_PORT:-5433}:5432"
    volumes:
      - postgres-data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", 'pg_isready -U "$${POSTGRES_USER}" -d "$${POSTGRES_DB}"']
      interval: 3s
      timeout: 3s
      retries: 15
  redis:
    image: redis:7-alpine
    command: ["redis-server", "--appendonly", "yes", "--appendfsync", "everysec", "--maxmemory-policy", "noeviction"]
    ports:
      - "127.0.0.1:${REDIS_PORT:-6379}:6379"
    volumes:
      - redis-data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 3s
      timeout: 3s
      retries: 15

volumes:
  postgres-data:
  redis-data:
```

Use the product's compatible Postgres/Redis versions. Include Postgres only if the application uses it. Store authoritative Redis data on a named volume with explicit persistence and eviction behavior. `everysec` is a local choice with a possible loss window. Choose production durability and backups from the dataset's recovery contract. Keep development dependencies on loopback when accessed by a host API, and use private Compose service names when the app also runs in Compose. Let Compose scope service and volume names to the project.

## Start and stop entrypoints

Commit `.env.example` with `POSTGRES_USER`, `POSTGRES_PASSWORD`, `POSTGRES_DB`, `POSTGRES_PORT`, `DATABASE_URL`, `REDIS_PORT`, and `REDIS_URL` as applicable. Document expected local host/ports. For this sample, `DATABASE_URL` uses `127.0.0.1:5433` and `REDIS_URL` uses `127.0.0.1:6379`. Keep `.env.development` out of Git and the Docker build context. Set separate container initialization and application values explicitly. Parse a URL in Node if needed, including escaped credentials, IPv6 and options. Do not source dotenv files as Bash code.

```bash
#!/usr/bin/env bash
# start-local.sh, stored at the repository root
set -euo pipefail
root="$(cd -- "$(dirname -- "${BASH_SOURCE[0]}")" && pwd)"
cd "$root"
env_file=.env.development
[[ -f "$env_file" ]] || { printf '%s\n' 'Create .env.development from .env.example.' >&2; exit 1; }

docker compose --project-directory "$root" --env-file "$env_file" -f docker/compose.yaml up --wait --wait-timeout 60
npm run db:migrate:local
exec npm run start:dev
```

```bash
#!/usr/bin/env bash
# stop-local.sh, stored at the repository root
set -euo pipefail
root="$(cd -- "$(dirname -- "${BASH_SOURCE[0]}")" && pwd)"
cd "$root"
exec docker compose --project-directory "$root" --env-file .env.development -f docker/compose.yaml down --timeout 15
```

The example assumes a Compose release with `up --wait`. The Compose `--env-file` only configures Compose. The host scripts must also load `.env.development` through their own application and migration config. For example, use `nest start --watch --env-file .env.development` for `start:dev` on a compatible Nest CLI. For Prisma, define `db:migrate:local` as `NODE_ENV=development prisma migrate deploy` and load `.env.development` via `loadEnvFile` from `node:process` in the local branch of `prisma.config.ts` on supported Node versions. Keep release migrations on separately injected runtime config. Omit the migration command for a backend without schema migrations. A migration failure stops startup.

Press Ctrl+C in the API terminal before running `./stop-local.sh` for the Compose dependencies. A port conflict is a startup error to resolve with the local configuration. Keep the stop command scoped to the Compose project and retain named volumes. [Compose CLI](https://docs.docker.com/reference/cli/docker/compose/) · [Startup readiness](https://docs.docker.com/reference/cli/docker/compose/up/) · [Shutdown](https://docs.docker.com/reference/cli/docker/compose/down/)

## Release image and runtime roles

Build with the lockfile, copy compiled runtime assets including i18n catalogs, and match build/runtime Node, libc and architecture for native dependencies such as Argon2 and ORM engines. For a multi-arch release, build or generate native artifacts for each target. Keep secrets out of the build context and inject them at runtime. Run Node as PID 1 or through the platform init. Use the actual emitted paths for API and, when present, worker. A worker should use its own application module graph and enable only its assigned job role. [Worker and scheduler setup](background-work.md#apiworker-separation)

```bash
# Example CI build and local smoke run. NODE_IMAGE is a reviewed pinned base image.
docker build --build-arg NODE_IMAGE="$NODE_IMAGE" -t backend:smoke .
docker run --rm --env-file .env.smoke -p 127.0.0.1:8080:8080 backend:smoke
# When a worker is needed, use the same image with an explicit worker command.
docker run --rm --env-file .env.worker.smoke backend:smoke node dist/worker.js
```

In smoke env files, use DB/Redis addresses reachable from the container. In production, deploy the same immutable digest for API and worker when they share build inputs, supply role-specific configuration, and run schema migrations once before any rollout step that needs them. API readiness checks HTTP and required services. Worker readiness checks its consumer or scheduler and required services. API and worker close resources under SIGTERM. Use [CI/CD](ci-cd.md#migration-and-deployment-order) for the release sequence.

Verify `bash -n` for both scripts and `docker compose ... config --quiet` using example values. In an isolated Compose project, exercise readiness, migrations, API boot, and stop/start with data still present. Build the actual release image and smoke-test the API command and worker command where applicable on the deployment architecture. If Docker is unavailable, report which runtime checks are pending.
