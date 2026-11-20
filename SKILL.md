---
name: modular-backend
description: Build, extend, refactor, or fix NestJS and Node.js backends using Fastify for HTTP, modular controller/service/repository boundaries, efficient queries, explicit Redis roles, auth/user/device workflows, reliable jobs, and CI/CD. Use for new backend infrastructure, backend features, and existing backend revisions; not frontend-only tasks or permission to rewrite unrelated code.
license: MIT
---

# Modular Backend

Complete the requested behavior with the smallest readable change. Apply these conventions to new backends and the affected code in existing projects. Do not turn a focused fix into a whole-project migration.

## Load the relevant references

| Task | Read |
| --- | --- |
| New module, structural refactor, DTOs, errors, dependency injection | [Architecture](references/architecture.md) |
| Queries, indexes, transactions, Redis, cache/storage behavior | [Data and Redis](references/data-and-redis.md) |
| User, auth, sessions, OAuth, device, permissions | [Identity](references/identity.md) |
| Bootstrap, config, rate limits, logs, health, verification | [Operations](references/operations.md) |
| Queues, workers, schedules, webhook idempotency, resource limits | [Background work](references/background-work.md) |
| CI checks, containers, migrations, releases, rollback | [CI/CD](references/ci-cd.md) |

Read only what the task needs. Examples express responsibilities and invariants; adapt imports, names, schemas, framework versions, ORM, and Redis client to the repository. They are not a standalone application to paste wholesale. These instructions use portable Markdown and standard project tools; no provider-specific agent APIs, MCP services, paid integrations, shell aliases, or personal repositories are required. Follow the host agent's permission model and the user's requested scope.

## Establish scope before editing

Read repository instructions, package manifest, bootstrap, affected modules and callers, relevant schemas, Redis usage, and tests. Use config definitions and example variable names rather than reading secrets. Briefly identify affected layers/files, observable behavior, and necessary checks. Resolve routine choices from existing code; ask only when a missing product decision changes behavior or data ownership.

### New backend

- Use Fastify for all new HTTP backends. Default to NestJS and TypeScript with an explicit `FastifyAdapter` from `@nestjs/platform-fastify`. For Node.js HTTP without Nest, use `fastify` directly with the same module boundaries. Workers without HTTP use their existing worker entry point.
- Implement only required modules/features. Auth/user/device workflows are reusable guidance, not permission to add account features to a service that does not need them.
- Plan Redis as baseline infrastructure with a useful workload role. Use the project's existing connection or an approved service interface. If an external client is required under the package policy, surface that requirement before selecting or adding it.
- Use Node.js built-ins and official NestJS modules first. Install the Nest core, `@nestjs/platform-fastify`, required peers, and necessary build tooling. Direct `fastify` is part of the approved HTTP stack when the application imports it. Add official optional Nest modules only for required features. Other new third-party dependencies, including optional peers and Fastify plugins, require explicit user approval. Reuse existing integrations in revisions. See [Operations](references/operations.md#package-policy).
- Start HTTP setup in `main.ts` with a `bootstrap()` function. Extract shared setup into `configureHttp(app, config)` when another entry point or integration test needs it. Keep external service startup and shutdown in the owning providers.
- Deliver connected bootstrap, validated config, providers, required migrations, local setup, and the requested APIs. Do not stop at disconnected scaffold files.

### Existing backend / revision

- Find all callers before changing a shared function or moving files: services, controllers, jobs, guards, exports, mocks, fixtures, and response consumers.
- Preserve unrelated changes, public contracts, token/storage behavior, validation, ownership filters, and established naming. Restrict cleanup to the task's path.
- Use Fastify when the task sets up or replaces the HTTP stack. An existing service on a different adapter needs a scoped migration to Fastify. Preserve its adapter during unrelated changes and surface the migration requirement when relevant. Verify status codes, middleware, cookies, uploads, and signed webhook bodies where the migration touches them.
- Extract queries to concrete repositories when changing the relevant service. Preserve transaction context, locks, error handling, return shape, soft-delete and tenant scopes.
- For an infrastructure setup or refactor, apply the same package policy. Preserve existing integrations unless the task changes them. Resolve dependencies required for the requested infrastructure before adding packages.
- Do not silently remove fallback, switch session storage, change token lifetimes, or invalidate all sessions as cleanup. Plan compatibility and data/key migrations for requested transitions.
- Produce necessary migrations; do not delete live data or execute production migrations incidentally.

## Architecture rules

1. **Use the full layout for each feature under `src/modules/`.** Every feature module includes `dto/`, a controller, a service, a repository, and `<feature>.module.ts`. Keep a single controller, service, or repository at the module root. When a layer gets multiple files, group it in `controllers/`, `services/`, or `repositories/`. Use `errors/` for module-specific errors when needed. Put supporting providers under infrastructure or jobs under workers when they have no feature API or data access. On a scoped revision preserve existing paths unless a module layout change is part of the task.
2. **Controller handles HTTP; service decides; repository accesses data.** No business logic, ORM queries, or Redis commands in controllers. Services own rules and orchestration. Repositories own queries and Redis operations, including conditional writes enforcing atomic invariants.
3. **Compose practically.** Cross-module business actions call the owning service. Direct use of an exported repository is allowed for data-only work or shared transactions. Do not create pass-through service methods for appearances.
4. **Keep transaction ownership close to the work.** Service can span repositories with one passed transaction client; repository can own a self-contained atomic operation. Every participant uses that client. No generic Unit of Work or transaction framework.
5. **Write little, readable code.** Reuse helpers, standard APIs, and installed dependencies. Avoid speculative options, wrappers, base classes, generic repositories, and one-use abstractions. Remove redundant validation and catch-and-rethrow; retain boundary validation, authorization, concurrency, and protection against data loss.
6. **Reduce actual work.** Minimize queries, round trips, fields, rows, and intermediate allocations. Parallel per-item queries remain N+1. Balance time and space for the workload; do not promise both absolute minima or weaken password hashing for speed.
7. **Justify indexes.** Link each index to a real query or integrity constraint. Inspect primary/unique/composite indexes before adding more. Do not add speculative duplicates or remove indexes based only on schema appearance.
8. **Assign Redis a role.** Distinguish primary data, prepared read models, and disposable caches. Define ownership, population/update, TTL, miss, outage, and recovery. No automatic catch-to-DB fallback. Cache misses and connection failures are separate; choose policy from the data reference.
9. **Share only shared code.** Common helpers belong in utils/common, infrastructure in libs, shared settings in config. Keep module-only helpers local. Centralize Redis keys and TTLs.
10. **Own and type errors.** Domain errors live in the module and use the existing typed registry/AppException/filter. Preserve public error envelopes; keep internal queries, stack traces, secrets, and provider details out of responses.

## Complete and verify

Update necessary callers, DTOs, error definitions, providers/exports, config, schemas/migrations, and tests. Implement important failures as well as success.

New non-trivial logic needs a small behavior test or assert-based check. Verify affected invariants: ownership, query count, rollback, token expiry/reuse/concurrency, Redis miss versus outage, or atomic updates. Trivial moves do not need implementation-mirroring tests. Run the relevant build/typecheck and required project checks without unrelated autofixes. Do not claim performance, integration coverage, or production/security verification you did not perform.

For requested delivery infrastructure, make CI a real gate before release. Keep untrusted pull requests unprivileged, build an immutable artifact, run necessary migrations once, wait for API/worker readiness, smoke-test, and preserve rollback information. Read CI/CD before changing workflows or container/release behavior; do not deploy or publish merely because a workflow was authored.

## Portability and maintenance

Examples are generic, reconstructed patterns with no private application identifiers or source dependencies. Do not infer production suitability from an example alone. Check installed types and official documentation for version-sensitive APIs; do not upgrade majors to fit a snippet. Prefer the project's existing CI provider, deployment platform, logger, queue, and ORM over adding parallel tools.
