# CI/CD and release engineering

## Contents

- Scope and release contract
- Pull request CI example
- Integration tests and containers
- Migration and deployment order
- Trust, credentials, rollback, and verification

## Scope and release contract

Use the repository's existing CI provider and deployment platform. GitHub Actions and Docker below illustrate the same principles available elsewhere; do not impose AWS, Kubernetes, GitHub, or a new deployment service. Authoring a workflow is not authorization to publish/deploy, change cloud IAM, or write secrets. Respect the host's permissions and any already authorized deployment scope.

Before changing CI/CD, inspect scripts and lockfiles, supported runtime, generated clients, container build, migration history, release commands, environment boundaries, API/worker roles, readiness and rollback. Preserve working conventions unless they cause the requested problem.

Define the small release contract in project terms:

1. An unprivileged PR path installs locked dependencies and runs relevant non-mutating lint/type checks, tests and build.
2. Trusted release commits pass equivalent required checks before publishing an immutable artifact.
3. Build once per target platform and promote the same tested image/artifact digest between environments; runtime config is injected separately.
4. Apply necessary backward-compatible migrations once per environment release, then roll out compatible API/workers and wait for readiness.
5. Run bounded smoke checks, record the release/digest/migration state, and retain a known-good rollback target.

Do not deploy on every PR or hide failed checks with continue-on-error. Branch protection/required-status configuration is separate from workflow YAML; configure it only when authorized and otherwise state what remains to be set. Match the project's release branches rather than assuming main exists.

## Pull request CI example

This complete workflow example assumes npm/package-lock.json, an exact supported Node version in .nvmrc, scripts named lint:check/typecheck/test/build, and GitHub-hosted Linux runners compatible with the pinned actions. Adapt to existing paths/monorepo/package manager; define real scripts rather than adding --if-present to skip missing gates. Lint must not run --fix or rewrite files. Generated clients must be available before typecheck, via the existing postinstall or an explicit generation step.

```yaml
name: Backend CI
'on':
  pull_request: {}
  push:
    branches: [main]
  workflow_dispatch: {}

permissions:
  contents: read

concurrency:
  group: backend-ci-${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

jobs:
  verify:
    runs-on: ubuntu-latest
    timeout-minutes: 15
    steps:
      - uses: actions/checkout@3d3c42e5aac5ba805825da76410c181273ba90b1 # v7.0.1
        with:
          persist-credentials: false
      - uses: actions/setup-node@820762786026740c76f36085b0efc47a31fe5020 # v7.0.0
        with:
          node-version-file: .nvmrc
          cache: npm
      - run: npm ci
      - run: npm run lint:check
      - run: npm run typecheck
      - run: npm test
      - run: npm run build
```

Action commits were verified in [the official actions reusable workflow](https://github.com/actions/reusable-workflows/blob/d2e5e0ce44e51e9dec1ac8584fc0756e851e0241/.github/workflows/basic-validation.yml). Recheck supported versions/runner requirements when applying or updating; a pinned hash is immutable, not permanently current. Keep updates reviewable and verify the hash belongs to the intended upstream action. Prefer existing dependency-update automation over a new custom updater.

Use the lockfile's immutable/frozen mode for pnpm/yarn equivalents. Cache the package-manager download store with lockfile/runtime keys, not an untrusted mutable node_modules artifact. Do not share writeable privileged caches with untrusted PR code. Installing dependencies may execute lifecycle hooks; PR runners get no deployment credentials or production secrets.

## Integration tests and containers

For changed query/transaction/Redis invariants, add integration checks using disposable isolated DB/Redis services, migrations and deterministic fixtures. Match production-relevant versions/features without connecting to production. Wait for service readiness; a running container is not proof it accepts requests. Bound test durations and clean up resources. A mock test does not establish DB rollback or Redis atomicity.

```sh
# Example test job after locked dependency install and isolated service startup.
# These project scripts must exist and use test-only environment configuration.
npm run db:migrate:test
npm run test:integration
```

Keep database-dependent tests separate from fast unit checks where useful. Require integration gates for the relevant release; do not quietly skip them when setup fails. Do not make every docs-only PR run an expensive platform matrix.

Container example for an npm/Nest project, with .dockerignore excluding secrets/unneeded files. NODE_IMAGE must be a reviewed compatible Node base image reference pinned by digest; no version is universally prescribed. Build and runtime use the same image/platform so native dependencies match.

```dockerfile
# syntax=docker/dockerfile:1
ARG NODE_IMAGE
FROM ${NODE_IMAGE} AS build
WORKDIR /app
COPY . .
RUN --mount=type=cache,target=/root/.npm npm ci
RUN npm run build

FROM build AS production-deps
RUN npm prune --omit=dev --ignore-scripts

FROM ${NODE_IMAGE} AS runtime
ENV NODE_ENV=production
WORKDIR /app
COPY --from=build --chown=node:node /app/package.json ./package.json
COPY --from=production-deps --chown=node:node /app/node_modules ./node_modules
COPY --from=build --chown=node:node /app/dist ./dist
USER node
CMD ["node", "dist/main.js"]
```

This layout copies source before install so source/schema-dependent install hooks work; optimize manifest/schema layers only after inspecting actual hooks. BuildKit's npm cache avoids repeated downloads. Add required non-code runtime assets/generated artifacts explicitly; verify the actual output entrypoint, ESM/CJS behavior and ORM native/runtime requirements. Do not copy .env, private keys, service-account files or deployment credentials into any image stage/context. Do not add a second process manager by default.

```text
# .dockerignore starter; extend for the actual repository.
.git
node_modules
dist
coverage
.env
.env.*
!.env.example
*.pem
*.key
*.p8
*-credentials.json
```

Filename globs are not a complete secret policy; inspect intended build inputs and exclude actual secret filenames. A secret removed in a later layer still exists in an earlier layer. Use runtime secret injection or build-secret mounts when genuinely required, never ARG/ENV for build credentials. Supply production values via the deployment secret store; public workflows never cat environment files into outputs or logs.

Build only needed architectures. Do not copy host-compiled native binaries/Prisma engines into a different target architecture. If cross-compilation is necessary, use pinned upstream sources and verify native artifact architecture. Test the built runtime image and API/worker entrypoints, not only a builder's test suite. Capture the pushed digest and deploy that exact digest, not latest or a tag that can be overwritten.

## Migration and deployment order

Keep migrations in version control. Use the ORM's locked deployment command, not a dev migration or schema-push command against production. With the project's PrismaClient-era CLI, a migration job may use:

```sh
# Run from the locked release tooling environment with the intended DB secret.
# No ad hoc downloads of an unpinned CLI.
./node_modules/.bin/prisma migrate deploy
```

Match the installed major/schema/config path. The migration CLI may be omitted from a slim runtime image; use release tooling that actually includes the pinned CLI and migrations. Do not put this command in every API/worker replica's startup path. Serialize the environment release/migration operation and fail the rollout if the required migration fails. Application-level locking is not a reason to allow concurrent destructive releases.

For rolling deployment use expand -> compatible code -> bounded backfill/verification -> later contract. Old and new API/worker versions overlap, and delayed queue payloads may outlive a release. Do not rename/drop columns or remove payload fields before all relevant readers/writers are compatible. Backfills must be restartable and bounded. Review destructive changes, lock duration and backup/restore needs according to actual risk; not every additive migration needs a ceremony.

The deployment graph is provider-specific, but its dependency must be real:

```text
trusted commit
  -> verify (lint/typecheck/unit + relevant integration checks)
  -> build/test/publish artifact and capture digest
  -> environment release lock + scoped authentication
  -> one migration step when required
  -> deploy API/worker using that digest and runtime config
  -> readiness/stability wait
  -> bounded smoke tests
  -> record successful release or fail with actionable diagnostics
```

For a GitHub Actions deployment job, reuse the project's existing deployment command. This is a job fragment, not a complete drop-in workflow; verify/build-image jobs must actually provide the immutable digest and pass all required gates:

```yaml
deploy:
  needs: [verify, build-image]
  if: github.event_name == 'push' && github.ref == 'refs/heads/main'
  runs-on: ubuntu-latest
  environment: production
  permissions:
    contents: read
    id-token: write
  concurrency:
    group: backend-production-release
    cancel-in-progress: false
  timeout-minutes: 20
  # Steps: checkout release tooling, obtain scoped OIDC credentials,
  # migrate once if required, deploy needs.build-image.outputs.digest,
  # await readiness and smoke-test. Implement real platform steps.
```

Do not install this fragment as a successful empty deployment. Complete all platform steps and outputs when implementing the actual task. Separate cancelable CI concurrency from production release serialization. Provider workflow queues may not guarantee FIFO; prevent an older commit superseding a newer successful release, and use the platform's release locking/order controls when required. Honor existing protected-environment/reviewer rules; do not invent manual approvals when the repository/user has not requested them.

## Trust, credentials, rollback, and verification

- Untrusted PR code does not receive write tokens, production secrets, cloud roles or privileged persistent runners. Do not run an untrusted head checkout in pull_request_target or equivalent privileged context. Scope permissions per job.
- Prefer short-lived workload identity/OIDC when the selected provider supports it. Scope trust to the intended repository, branch/environment, audience and workflow identity; do not use a broad wildcard trust merely to get a login working. Verify current claim format. Existing credential policy changes remain within task authorization.
- Treat PR titles, commit messages and other event text as untrusted data. Pass values through a step environment/structured argument; never interpolate them directly into executable shell source. Avoid logging secrets through outputs, command echoing, notifications or upload artifacts. Do not add Slack/email notifications unless requested.
- Separate build-time and runtime configuration. Promoting the same image means no environment-specific build secrets baked into it. Record commit, digest, schema version and rollout status without secret values.
- Wait for each deployed role to be healthy/ready. Smoke-test the real route prefix, required prepared data and a representative safe request. Mutating smoke checks require isolated test identities and cleanup.
- Retain the previous known-good artifact and configuration. Application rollback must remain compatible with the expanded schema and in-flight jobs. Do not automatically reverse a destructive schema migration or assume rolling back the image restores lost data.
- Test recovery from failed checks, failed migration, partial rollout and failed smoke checks as relevant. Treat local YAML parsing as syntax validation only; it is not evidence that cloud permissions, registry access, image build or production deployment succeeded.

Sources: [GitHub workflow security](https://docs.github.com/en/actions/reference/security/secure-use), [OIDC](https://docs.github.com/en/actions/concepts/security/openid-connect), [deployment environments](https://docs.github.com/en/actions/concepts/workflows-and-actions/deployment-environments), [Prisma migration deployment](https://docs.prisma.io/docs/cli/migrate/deploy), [expand and contract](https://www.prisma.io/docs/guides/database/data-migration).
