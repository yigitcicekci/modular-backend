---
name: modular-backend
description: Build, extend, refactor, or fix modular NestJS and Node.js backends with Fastify, AppBuilder, Helmet, localized errors, Redis storage, rate limiting, and cursor pagination. Apply efficient queries, JWT and device auth, request guards, scheduled jobs, Docker, and CI/CD to new backends and existing systems.
license: MIT
---

# Modular Backend

Complete the requested behavior with the smallest readable change. Apply these conventions to new backends and the affected code in existing projects. Do not turn a focused fix into a whole-project migration.

## Load the relevant references

| Task | Read |
| --- | --- |
| New module, structural refactor, DTOs, dependency injection | [Architecture](references/architecture.md) |
| Error contracts, validation failures, global filters, i18n | [Errors and i18n](references/errors-and-i18n.md) |
| Queries, indexes, transactions, Redis, cache/storage behavior | [Data and Redis](references/data-and-redis.md) |
| User, auth packages, sessions, OAuth, devices, fingerprints, password/IP hashing | [Identity](references/identity.md) |
| Header and JWT guards, middleware, route and parameter decorators | [Request boundary](references/request-boundary.md) |
| Bootstrap, config, rate limits, logs, health, verification | [Operations](references/operations.md) |
| Queues, workers, schedules, webhook idempotency, resource limits | [Background work](references/background-work.md) |
| Dockerfile, local Compose, start/stop Bash scripts, API/worker commands | [Docker](references/docker.md) |
| CI checks, containers, migrations, releases, rollback | [CI/CD](references/ci-cd.md) |

For a new backend, foundation overhaul, or request to fix missing baseline components, read Architecture, Operations, Errors and i18n, Request boundary, Docker, and Data and Redis before editing. Read other references as the task needs them. Adapt examples to the installed versions and actual domain. Use portable Markdown and standard project tools. Follow the host agent's permission model.

## Establish scope before editing

Read repository instructions, package manifest, bootstrap, affected modules and callers, relevant schemas, Redis usage, and tests. Use config definitions and example variable names for discovery.

**New backend.** Apply the foundation and architecture rules directly. Implement the connected baseline and use the relevant references for requested product flows. Ask for missing product decisions that affect behavior or data ownership. The skill already sets the baseline engineering choices.

**Existing backend.** Start with a read-only [architecture assessment](references/architecture.md#existing-backend-assessment). Present the current architecture, the proposed target, and which components to keep, change, add, or defer. **"Revise this backend according to modular-backend"** covers the whole backend. When a request specifies the standard without selecting specific changes, ask in one concise question which proposals to apply and wait for the decision. If the user already explicitly requested a specific change, analyze and share the relevant architecture first, proceed with that change, and ask before taking on further architectural changes. An analysis-only request ends with the assessment. After the user selects the scope, implement it and run the applicable acceptance checks.

For a request naming a feature, endpoint, or bug, limit the assessment to that path and its affected callers. Use the closest existing module that follows these rules as the model for naming, file placement, DTOs, errors, and transaction style.

### New backend, foundation overhaul, or baseline remediation

- **AppBuilder and Helmet are required for HTTP.** Keep `src/main.ts` thin and put HTTP construction in `src/builder/app.builder.ts`. Use an explicit Nest `FastifyAdapter` or direct `fastify` for Node.js HTTP without Nest. Register compatible `@fastify/helmet` before initialization or listening. Bind the real builder in HTTP integration tests. Workers without HTTP use their own entry point.
- **Localize HTTP errors in one global filter or Fastify handler.** Use module error definitions with `code`, `status`, and `messageKey`, a typed registry, and a shared exception. Return `{ status, code, message }` for new HTTP contracts. `status` matches the HTTP status, `code` stays stable across languages, and `message` comes from i18n. For existing APIs, preserve published fields until a compatible migration is agreed.
- **Redis infrastructure and a real Redis data path are required.** Add validated connection config, centralized Redis key and TTL files, one shared Redis provider, and a readiness probe. Assign Redis at least one useful data family for which it is the source of truth. Implement its real reads and writes through a domain repository. A connection, health probe, or SQL response cache alone does not satisfy this rule. Use a suitable existing product flow such as published client configuration, live ranking/state, or shared throttling. Where the product has client-config or version policy, store and serve its published document from Redis. Identify a missing product decision if no legitimate data family can be chosen from scope.
- **Generate Redis keys from shared templates.** Keep `[placeholder]` templates in `redis-keys.config.ts`, TTLs in `redis-ttl.config.ts`, and `generateRedisKey`/`generateRedisKeys` in `src/utils/redis-key.util.ts`. Have the shared provider apply the configured namespace once. Use this path in real readers, writers and jobs. Preserve the existing key format during revisions unless the user selects a planned migration. See [Redis key examples](references/data-and-redis.md#keys-ttls-and-command-examples).
- **Redis-owned and prepared reads stay on Redis.** Set the population, update, missing-key, expiry, outage, and recovery behavior per key family. Use typed absence for a genuine missing entity and typed unavailable for an outage or a required projection that is not ready. A request never queries SQL as a fallback for these roles. Use an explicit cache-aside exception only for an agreed disposable cache. Preserve an existing read contract on unrelated revisions.
- **Wire shared rate limiting for HTTP.** Register the enforcement guard or hook and configure a default budget for external routes. Add tighter Redis-backed budgets to auth, recovery, and expensive actions that exist. Use separate IP and normalized account counters for password login. Increment and expire atomically, respond with 429 and `Retry-After`, and return typed 503 when required enforcement is unavailable. Use an explicit policy for system route exemptions.
- **Paginate every growing collection exposed by HTTP.** Use a validated query DTO, bounded limit, and a stable keyset cursor containing every sort value plus a unique tie-breaker. Query `limit + 1` with the caller's scope and filters in the repository. Return `{ items, nextCursor }` for new lists. Calculate exact totals and use page-number offset only when the API contract needs them.
- **Wire the guards and decorators the API uses.** Use a typed header guard for a mobile/device contract, a JWT guard for protected JWT routes, and role, guest, or provider-specific guards where the policy needs them. Register the guards and consume their validated context using parameter decorators. Use Fastify request hooks or existing middleware for cross-cutting transport concerns.
- **Deliver a runnable Docker workflow.** Provide or reuse a Dockerfile, Compose for local dependencies, and project-scoped Bash start/stop commands. Wait for readiness and use durable volumes for authoritative data. Retain an equivalent established workflow on revisions. Run job schedules and consumers only when the workload warrants them.
- Implement only required modules/features. Auth/user/device workflows are reusable guidance, not permission to add account features to a service that does not need them.
- Use Node.js built-ins and official NestJS modules first. The approved baseline includes compatible `@fastify/helmet`, a single Redis client, and `nestjs-i18n` for Nest HTTP errors if no suitable i18n integration is installed. A new password-auth flow may use compatible `argon2` for Argon2id; use `node:crypto` for token digests and keyed fingerprints. Keep the existing working Redis client or default to `redis` for a new project. Use `fastify` directly where imported. Other new third-party packages, optional peers and Fastify plugins require explicit approval. See [Operations](references/operations.md#package-policy).
- Deliver connected bootstrap, validated config, providers, required migrations, local setup, and the requested APIs. Do not stop at disconnected scaffold files.

### Existing backend / revision

- Find all callers before changing a shared function or moving files: services, controllers, jobs, guards, exports, mocks, fixtures, and response consumers.
- Preserve unrelated changes, public contracts, token/storage behavior, validation, ownership filters, and established naming. Restrict cleanup to the task's path.
- When the task sets up, replaces, or repairs the HTTP foundation, bring Fastify, AppBuilder and Helmet into scope together. Preserve the adapter and bootstrap during unrelated feature changes. When migrating to Fastify, verify status codes, middleware, cookies, uploads, and signed webhook bodies where relevant.
- Extract queries to concrete repositories when changing the relevant service. Preserve transaction context, locks, error handling, return shape, soft-delete and tenant scopes.
- When changing a list or limiter, preserve the existing public cursor and response contract and tighten validation within scope. Trace rate-limit metadata to a running guard or hook. Keep actor and tenant scoping in the data query. Plan incompatible cursor or limiter key changes explicitly.
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
10. **Own and type errors.** Define domain errors under the module's `errors/` directory. Throw the registered `ErrorCode` through `AppException` and translate its `messageKey` at the HTTP boundary. Keep internal queries, stack traces, secrets, and provider details out of responses.

## Complete and verify

Update necessary callers, DTOs, error definitions, providers/exports, config, schemas/migrations, and tests. Implement important failures as well as success.

For foundation work, follow the [acceptance checks](references/operations.md#foundation-acceptance-checks). Trace the production entry point through the builder and one real feature through its Redis repository. Verify Helmet, the localized HTTP error contract, Redis initialization and shutdown, and miss versus outage. Verify 429 and 503 on real limited routes, and test stable pagination on each new list contract. Report any baseline item that is missing or unverified.

New non-trivial logic needs a small behavior test or assert-based check. Verify affected invariants: ownership, query count, rollback, token expiry/reuse/concurrency, Redis miss versus outage, or atomic updates. Trivial moves do not need implementation-mirroring tests. Run the relevant build/typecheck and required project checks without unrelated autofixes. Do not claim performance, integration coverage, or production/security verification you did not perform.

For requested delivery infrastructure, make CI a real gate before release. Keep untrusted pull requests unprivileged, build an immutable artifact, run necessary migrations once, wait for API/worker readiness, smoke-test, and preserve rollback information. Read CI/CD before changing workflows or container/release behavior; do not deploy or publish merely because a workflow was authored.

## Portability and maintenance

Examples are generic, reconstructed patterns with no private application identifiers or source dependencies. Do not infer production suitability from an example alone. Check installed types and official documentation for version-sensitive APIs; do not upgrade majors to fit a snippet. Prefer the project's existing CI provider, deployment platform, logger, queue, and ORM over adding parallel tools.
