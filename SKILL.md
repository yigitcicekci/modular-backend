---
name: modular-backend
description: Build, extend, refactor, or fix NestJS and Node.js backends with Fastify, AppBuilder, Helmet, modular controller/service/repository boundaries, Redis storage, shared rate limiting, and cursor pagination. Apply efficient queries, identity, background work, and CI/CD to new backends and existing systems.
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

For a new backend, foundation overhaul, or request to fix missing baseline components, read Architecture, Operations, and Data and Redis before editing. Read other references as the task needs them. Adapt examples to the installed versions and actual domain. Use portable Markdown and standard project tools. Follow the host agent's permission model.

## Establish scope before editing

Read repository instructions, package manifest, bootstrap, affected modules and callers, relevant schemas, Redis usage, and tests. Use config definitions and example variable names rather than reading secrets. Briefly identify affected layers/files, observable behavior, and necessary checks. Resolve routine choices from existing code; ask only when a missing product decision changes behavior or data ownership.

A request such as **"Revise this backend according to modular-backend"** with no narrower target is a backend-wide revision. Apply both the foundation and existing-backend rules. Inventory the HTTP bootstrap, feature modules, database and Redis paths, and delivery checks. Identify gaps against the applicable references, fix them within the requested scope, and run the foundation acceptance checks. Report any gap that requires an unresolved contract, data migration, or ownership decision. If the request names a specific feature, endpoint, or bug, apply the rules to that path and its affected callers.

Use the closest existing module that follows these rules as the model for naming, file placement, DTOs, errors, and transaction style.

### New backend, foundation overhaul, or baseline remediation

- **AppBuilder and Helmet are required for HTTP.** Keep `src/main.ts` thin and put HTTP construction in `src/builder/app.builder.ts`. Use an explicit Nest `FastifyAdapter` or direct `fastify` for Node.js HTTP without Nest. Register compatible `@fastify/helmet` before initialization or listening. Bind the real builder in HTTP integration tests. Workers without HTTP use their own entry point.
- **Redis infrastructure and a real Redis data path are required.** Add validated connection config, centralized Redis key and TTL files, one shared Redis provider, and a readiness probe. Assign Redis at least one useful data family for which it is the source of truth. Implement its real reads and writes through a domain repository. A connection, health probe, or SQL response cache alone does not satisfy this rule. Use a suitable existing product flow such as published client configuration, live ranking/state, or shared throttling. Where the product has client-config or version policy, store and serve its published document from Redis. Identify a missing product decision if no legitimate data family can be chosen from scope.
- **Redis-owned and prepared reads stay on Redis.** Set the population, update, missing-key, expiry, outage, and recovery behavior per key family. Use typed absence for a genuine missing entity and typed unavailable for an outage or a required projection that is not ready. A request never queries SQL as a fallback for these roles. Use an explicit cache-aside exception only for an agreed disposable cache. Preserve an existing read contract on unrelated revisions.
- **Wire shared rate limiting for HTTP.** Register the enforcement guard or hook and configure a default budget for external routes. Add tighter Redis-backed budgets to auth, recovery, and expensive actions that exist. Use separate IP and normalized account counters for password login. Increment and expire atomically, respond with 429 and `Retry-After`, and return typed 503 when required enforcement is unavailable. Use an explicit policy for system route exemptions.
- **Paginate every growing collection exposed by HTTP.** Use a validated query DTO, bounded limit, and a stable keyset cursor containing every sort value plus a unique tie-breaker. Query `limit + 1` with the caller's scope and filters in the repository. Return `{ items, nextCursor }` for new lists. Calculate exact totals and use page-number offset only when the API contract needs them.
- Implement only required modules/features. Auth/user/device workflows are reusable guidance, not permission to add account features to a service that does not need them.
- Use Node.js built-ins and official NestJS modules first. The approved baseline includes compatible `@fastify/helmet` and a single Redis client. Keep the existing working client, including ioredis if used. Default to `redis` (node-redis) for a new project without one. Use `fastify` directly where imported. Other new third-party packages, optional peers and Fastify plugins require explicit approval. See [Operations](references/operations.md#package-policy).
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
10. **Own and type errors.** Domain errors live in the module and use the existing typed registry/AppException/filter. Preserve public error envelopes; keep internal queries, stack traces, secrets, and provider details out of responses.

## Complete and verify

Update necessary callers, DTOs, error definitions, providers/exports, config, schemas/migrations, and tests. Implement important failures as well as success.

For foundation work, follow the [acceptance checks](references/operations.md#foundation-acceptance-checks). Trace the production entry point through the builder and one real feature through its Redis repository. Verify Helmet, Redis initialization and shutdown, and miss versus outage. Verify 429 and 503 on real limited routes, and test stable pagination on each new list contract. Report any baseline item that is missing or unverified.

New non-trivial logic needs a small behavior test or assert-based check. Verify affected invariants: ownership, query count, rollback, token expiry/reuse/concurrency, Redis miss versus outage, or atomic updates. Trivial moves do not need implementation-mirroring tests. Run the relevant build/typecheck and required project checks without unrelated autofixes. Do not claim performance, integration coverage, or production/security verification you did not perform.

For requested delivery infrastructure, make CI a real gate before release. Keep untrusted pull requests unprivileged, build an immutable artifact, run necessary migrations once, wait for API/worker readiness, smoke-test, and preserve rollback information. Read CI/CD before changing workflows or container/release behavior; do not deploy or publish merely because a workflow was authored.

## Portability and maintenance

Examples are generic, reconstructed patterns with no private application identifiers or source dependencies. Do not infer production suitability from an example alone. Check installed types and official documentation for version-sensitive APIs; do not upgrade majors to fit a snippet. Prefer the project's existing CI provider, deployment platform, logger, queue, and ORM over adding parallel tools.
