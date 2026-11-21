# Infrastructure and operational behavior

## Contents

- Bootstrap and configuration
- Package policy
- Foundation acceptance checks
- Shared rate limiting
- Logs and health
- Integrations, verification, and examples of applying the skill

## Bootstrap and configuration

For new HTTP backends and foundation work, create `src/builder/app.builder.ts`. Export `buildApp()` or use the project's existing AppBuilder class. The builder creates and configures the Fastify app and returns it without listening. Use the same builder in `src/main.ts` and HTTP integration tests. Wrap the HTTP options in one validated config object. Register global guards, filters, and interceptors through Nest module providers such as `APP_GUARD`, `APP_FILTER`, and `APP_INTERCEPTOR` when they need dependency injection. Register each concern once. External service providers own their startup and cleanup.

```ts
// src/builder/app.builder.ts
import helmet from '@fastify/helmet';
import { NestFactory } from '@nestjs/core';
import { FastifyAdapter, type NestFastifyApplication } from '@nestjs/platform-fastify';
import { AppModule } from '../app.module';
import type { HttpConfig } from '../config/app.config';

export async function buildApp(config: HttpConfig) {
  const app = await NestFactory.create<NestFastifyApplication>(
    AppModule,
    new FastifyAdapter({ trustProxy: config.trustedProxies }),
    { abortOnError: false },
  );
  try {
    await app.register(helmet);
    app.setGlobalPrefix(config.apiPrefix, { exclude: config.prefixExclusions });
    app.enableCors({
      origin: config.corsOrigins.length ? config.corsOrigins : false,
      credentials: config.cookieAuth,
    });
    return app;
  } catch (error) {
    await app.close();
    throw error;
  }
}
```

```ts
// src/main.ts, inside bootstrap() after loading validated config.
const app = await buildApp(config);
try {
  app.enableShutdownHooks();
  await app.listen(config.port, config.host);
} catch (error) {
  await app.close();
  throw error;
}
```

Here `HttpConfig` includes a validated host, port, prefix, prefix exclusions, explicit CORS origin list, cookie-auth flag, and `false` or a trusted-proxy address list. Keep prefix exclusions in `src/config/route.config.ts` when shared. Enable signal hooks in the executable entry point. Its top-level catch logs safe diagnostics and leaves a nonzero exit status after cleanup. Bind request DTO validation through [feature pipes](architecture.md#dtocontrollerservicerepository-example) or the project's installed global validation. Keep unknown-field rejection for new DTOs and preserve established behavior on revisions.

For Node.js HTTP without Nest, use the same builder boundary with `fastify()` and `await app.register(helmet)`. Keep adapter-specific request/reply APIs in transport code. Verify cookies, status codes, multipart routes and raw webhook bodies when migrating adapters. Configure trusted proxies explicitly before using forwarded addresses in guards or rate limits. Keep Helmet defaults for API routes. If Swagger needs a CSP adjustment, scope it to the docs routes and test both the docs and API headers. [Nest Fastify adapter](https://docs.nestjs.com/techniques/performance#fastify-adapter) · [Nest Helmet integration](https://docs.nestjs.com/security/helmet#fastify)

Config values are parsed/validated at startup, then injected. Never silently accept malformed ports, TTLs, empty secrets, or default production credentials. Do not scatter process.env reads through services. Keep actual secrets outside source and logs; document needed variable names in the project's example environment file.

```ts
// example: config-integer
export function positiveInteger(value: string | undefined, name: string): number {
  if (!value || !/^[1-9]\d*$/.test(value)) {
    throw new Error(`${name} must be a positive integer`);
  }
  const number = Number(value);
  if (!Number.isSafeInteger(number)) throw new Error(`${name} is too large`);
  return number;
}
```

This small example is only for a new project without an established schema validator. Reuse the installed validator otherwise. Add actual domain bounds where needed (port <= 65535, policy-specific TTL maxima). Defaults must be intentional; zero may be valid for a different setting. For boolean text, parse explicit true/false; `Boolean('false')` is true.

Redis lifecycle belongs in its shared provider: validate config, register an error listener, use bounded connection/command behavior and reconnect policy, connect on initialization and close on shutdown. Choose offline queue behavior explicitly; an offline queue must not turn a supposed fast failure into unbounded waits. Match installed client APIs and avoid a Redis wrapper for each native command. Required Redis-backed features must not become silently successful while disconnected.

If the project uses OpenAPI, derive the contract from real routes and DTOs. Document authentication, success/error responses and pagination, and update affected generated clients/specs when the task changes them. Do not add a second manual contract that drifts from runtime validation. Publish administrative/debug routes or API documentation only according to the existing access policy; hiding a route from documentation is not authorization.

```ts
// Bootstrap fragment when the installed Nest Swagger integration is used.
const document = SwaggerModule.createDocument(app, swaggerConfig);
SwaggerModule.setup('docs', app, document);
```

Gate or protect docs according to the product's deployment policy; do not expose debug/test controllers in production by default.

When mobile clients send app version, platform, or device headers, validate and normalize them once in a shared client-context guard or decorator. Pass the resulting typed context to the owning service. The device header describes a client installation. Authorization still comes from the verified user and session. Keep explicit exceptions for liveness, provider webhooks, or public configuration routes that predate client bootstrap. Test those exceptions alongside protected routes and keep error codes stable when messages are localized.

## Package policy

For Nest HTTP, use `@nestjs/common`, `@nestjs/core`, `@nestjs/platform-fastify`, required framework peers, and compatible `@fastify/helmet`. For Node.js HTTP without Nest, install `fastify` and its compatible Helmet plugin. In Nest, declare `fastify` directly if application code imports it. Use `typescript` and `@types/node` as dev tooling when applicable, plus `@nestjs/cli` for `nest build`. Match installed majors and the lockfile. Helmet and one Redis client are baseline dependencies. Other Fastify plugins need a required feature and user approval.

Prefer built-in `fetch`, `AbortSignal.timeout()`, `node:crypto`, `Intl`, `node:test`, `node:assert/strict`, and `--env-file` on supported Node versions. Nest built-in `Parse*` pipes cover individual values. A small feature pipe can validate a DTO body. Use `ValidationPipe` from an existing project when its `class-validator` and `class-transformer` dependencies are already installed. Adding these packages requires explicit approval.

Official Nest packages such as `@nestjs/config`, `@nestjs/jwt`, `@nestjs/swagger`, `@nestjs/terminus`, `@nestjs/schedule`, and `@nestjs/testing` are feature choices. Verify their dependency requirements. Reuse the installed Redis client or select `redis` for a new project. Other database drivers, durable queue implementations, and provider SDKs require an existing integration or explicit approval. A compatible existing HTTP service can be called through `fetch` when its API meets the feature needs. Never hand-roll a Redis or database protocol, cryptographic token signing, or delivery guarantees to avoid a dependency.

See [Nest validation](https://docs.nestjs.com/techniques/validation), [Nest lifecycle](https://docs.nestjs.com/fundamentals/lifecycle-events), and [Node globals](https://nodejs.org/api/globals.html).

## Foundation acceptance checks

Run these against the relevant development or test environment before reporting a foundation complete. If infrastructure cannot run there, identify the specific check that remains.

| Requirement | Evidence |
| --- | --- |
| Fastify + AppBuilder + Helmet | The production entry point calls `buildApp()`. Tests build through it, call `app.init()`, await Fastify readiness, and use `inject()` on a real API route. Assert a Helmet header such as `x-content-type-options: nosniff` and the expected routing/CORS behavior. Close the app after testing. |
| Validated Redis configuration | Invalid or missing connection config fails clearly. Required Redis is probed with a deadline. After startup, readiness returns 503 on a Redis outage while liveness remains independent. |
| Real Redis data feature | Exercise a write or publication and then the actual API read. Check the Redis key with the shared key builder. Verify a true miss and an outage separately. Prove the read path has no SQL fallback and that data outlives an app restart when its lifecycle requires it. |
| Shared rate limiting | Hit the running HTTP route from two app instances sharing Redis and verify that they use one counter. Verify the first allowed requests, 429 with `Retry-After`, expiry and renewed access, explicit route exemptions, and typed 503 during a required Redis failure. Check forwarded address behavior with the actual proxy configuration. |
| Paginated growing lists | Verify default and maximum limits, rejection of malformed parameters, tied timestamps across two pages, a deleted cursor row, tenant or owner scope, and bounded query count. Verify no exact count is run unless requested by the contract. |
| Feature structure and package policy | Check each affected feature for DTO, controller, service, repository, and module. Check provider wiring and the lockfile. Confirm Helmet and the Redis client are used at runtime and avoid adding duplicate clients or optional integrations. |

## Shared rate limiting

Register active enforcement for external HTTP routes in foundation work. Define a coarse default IP budget and tighter route or action policies in typed shared config. Document exemptions for health or ops routes that run independently of Redis and use separate authentication and provider controls where needed. Nest can use a DI-registered `APP_GUARD` with route metadata or a verified existing throttler with explicit Redis storage. For direct Fastify, wire it in the appropriate request hook. A package declaration, a decorator, and unused Redis helpers do not enforce a limit.

Use a stable policy ID and Redis namespace. Derive client IP from Fastify `request.ip` after configuring trusted proxy addresses, and use an authenticated actor for user or tenant budgets. When authentication runs after a global guard, check actor-specific rules in a guard or service that runs after authentication. Bound key size and cardinality. For passwords, use **independent** counters for client IP and normalized account before password hashing, including unknown accounts. One combined IP-and-account key permits either side to rotate. Keep identifiers out of Redis keys by hashing, preferably with HMAC for guessable account identifiers. Apply separate action budgets after auth for expensive work such as uploads or external API calls.

This fixed-window Lua example atomically initializes count and TTL. Place it in the limiter's repository, not a controller. Policy window/maximum come from shared config; rejected attempts continue to count.

```lua
-- example: fixed-window-limit
local count = redis.call('INCR', KEYS[1])
if count == 1 then
  redis.call('PEXPIRE', KEYS[1], ARGV[1])
end
return {count, redis.call('PTTL', KEYS[1])}
```

```ts
// RateLimitRepository, node-redis. key is built by the shared config.
async increment(key: string, windowMs: number) {
  const [count, ttlMs] = await this.redis.run((redis) =>
    redis.eval(FIXED_WINDOW_LIMIT, {
      keys: [key], arguments: [String(windowMs)],
    }),
  ) as [number, number];
  if (ttlMs < 0) throw new AppException('RATE_LIMIT_UNAVAILABLE');
  return { count, ttlMs };
}
```

```ts
// Nest Fastify guard fragment. The limiter resolves policy and trusted actor.
const reply = context.switchToHttp().getResponse<FastifyReply>();
const { count, ttlMs } = await this.limiter.consume(policy, identity);
if (count > policy.limit) {
  reply.header('Retry-After', String(Math.max(1, Math.ceil(ttlMs / 1000))));
  throw new AppException('RATE_LIMITED');
}
return true;
```

Map `RATE_LIMITED` to HTTP 429 and preserve the header in the exception filter. Map an unavailable Redis counter to typed HTTP 503. A process-local fallback loses the shared budget when replicas diverge and is unsuitable for required enforcement. Initialize counters only via the atomic operation and do not extend their TTL on subsequent attempts. A fixed window allows bursts around boundaries. When tighter smoothing is required, reuse an existing supported sliding-window or token-bucket implementation. Check policy ordering, Redis sharing, and actual response headers with integration tests. [NestJS rate limiting](https://docs.nestjs.com/security/rate-limiting)

## Logs and health

Keep one request correlation ID across logs and error responses. Accept an upstream ID only after validating/bounding it, otherwise generate one. Log meaningful failures with duration, route and safe identifiers. Exclude or redact Authorization, cookies, passwords, access/refresh/reset tokens, secret config and sensitive body fields. Avoid recording entire request/response bodies by default. Auth/audit events use a small explicit allowlist of fields; do not serialize arbitrary input as metadata.

```ts
// Authentication audit event; no token/hash/password or raw request body.
this.logger.log({
  event: 'SESSION_REVOKED', requestId,
  actorUserId, targetSessionId, outcome: 'success',
});
```

Use the existing logger instead of a custom logging framework. Do not log the same exception at every layer. If audit persistence is security-critical, define durability/failure policy; a swallowed catch is not a durable audit trail. Best-effort operational telemetry can fail independently when stated.

Liveness checks process responsiveness. Readiness checks required dependencies and prepared data. Return HTTP 503 when a required check fails, and keep internal failure details out of the response. Wire timeouts so a health check cannot hang indefinitely. Verify that the load balancer or orchestrator calls the actual readiness and liveness routes.

```ts
// HealthService fragment; repositories/providers implement bounded checks.
async checkReady() {
  await Promise.all([this.database.ping(), this.redis.ping()]);
  // Add prepared-data readiness only if the requested product requires it.
  return { status: 'ok' as const };
}
```

Do not require SQL readiness for a Redis-only service. Prefer an installed health package where present. Invalidate readiness until mandatory initialization/prefetch is complete.

## Integrations, verification, and examples of applying the skill

Add email/notifications, queues, storage, uploads, billing, MFA or multi-tenancy only when needed. Shared integration providers manage transport; domain services own why/when to send or mutate. Keep external HTTP/email outside DB transactions. Use timeouts; retry only bounded, safe/idempotent operations. Never retry payments or non-idempotent writes blindly. Durable background work needs the existing queue/job mechanism or the smallest justified durable solution, not unawaited promises.

For requested schema changes, provide the migration and regenerate types when required. During revisions maintain compatibility for deployed consumers and existing records; do not erase data to make a new schema fit. Keep migrations/deployment separate from permission to modify production.

Choose verification by behavior, not line coverage:

| Change | Useful check |
| --- | --- |
| Thin controller/repository extraction | Existing public behavior, ownership predicate, transaction client preserved |
| Query batching | Query count stays bounded as records grow; same result mapping |
| Session rotation/reset | Real conditional write/rollback under concurrent attempts |
| Redis policy | Miss, failure, invalidation and absence are distinct |
| Device push ownership | Concurrent reassignments respect uniqueness and user boundaries |
| Migration | Existing records/clients remain readable; rollback/roll-forward story |

Use fakes for local branch behavior, not proof of real DB locking/Redis atomicity. Use the project's integration setup to prove those invariants. Do not run production writes or broad load tests without scope. State if required infrastructure is unavailable instead of claiming integration success. No new dependency just to test a few lines.

Example requests and expected scope:

- **New backend:** “Create a Nest API with email login and mobile devices.” Build connected required modules, DB schema, Redis roles, validation, local setup and auth/device behavior checks. Do not generate billing, MFA or an admin permission engine.
- **Revision:** “Move the profile query out of UserService.” Extract it to UserRepository; update callers/wiring/mocks; preserve response, permissions and transaction context. Do not switch ORM or change session storage.
- **Performance:** “Remove N+1 from library search.” Batch membership/card reads and map once; inspect query count; justify any index from the actual query. Do not claim Promise.all fixes N+1.
- **Auth:** “Make logout immediate.” Inspect the guard, token lifetime, authoritative session store and cache. Implement effective revocation with the required compatibility/data transition; deleting only a refresh token is insufficient for still-valid stateless access JWTs.

Sources: [Nest configuration](https://docs.nestjs.com/techniques/configuration), [Nest lifecycle](https://docs.nestjs.com/fundamentals/lifecycle-events), [OWASP logging](https://cheatsheetseries.owasp.org/cheatsheets/Logging_Cheat_Sheet.html), [Redis scripting](https://redis.io/docs/latest/develop/programmability/eval-intro/).
