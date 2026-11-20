# Infrastructure and operational behavior

## Contents

- Bootstrap and configuration
- Package policy
- Shared rate limiting
- Logs and health
- Integrations, verification, and examples of applying the skill

## Bootstrap and configuration

Start with one `bootstrap()` in `main.ts`. Use a `configureHttp(app, config)` function when entry points or integration tests share HTTP wiring. Keep a builder class for an existing setup with a real need for composable stages. HTTP setup covers prefix and exclusions, controlled CORS, validation, global errors, and shutdown hooks. Register guards, filters, and interceptors through Nest module providers such as `APP_GUARD`, `APP_FILTER`, and `APP_INTERCEPTOR` when they need dependency injection. The provider that owns an external connection also owns its startup and cleanup.

```ts
import { NestFactory } from '@nestjs/core';
import { FastifyAdapter, type NestFastifyApplication } from '@nestjs/platform-fastify';
import { AppModule } from './app.module';

// Inside bootstrap(), with validated config.
const app = await NestFactory.create<NestFastifyApplication>(
  AppModule,
  new FastifyAdapter(),
);
app.setGlobalPrefix(config.apiPrefix, { exclude: config.prefixExclusions });
app.enableCors({ origin: config.corsOrigins });
app.enableShutdownHooks();
await app.listen(config.port, config.host);
```

Bind request DTO validation through [feature pipes](architecture.md#dtocontrollerservicerepository-example) or the project's existing global validation. Match the existing rules on transformations and unknown fields during revisions. Use one startup error owner that reports safe diagnostics and terminates with a nonzero status. Exercise the real HTTP configuration in integration tests, including global providers, health route exclusions, and the selected adapter.

For Node.js HTTP without Nest, initialize `fastify()` directly. Keep adapter-specific request/reply APIs at the transport boundary. When migrating an existing HTTP stack to Fastify, check middleware, response serialization and status, CORS, cookies, multipart routes, and webhook body handling as applicable. Use the same adapter in production and HTTP integration tests. Fastify's `inject()` can exercise routes without opening a listening socket. [Nest Fastify adapter](https://docs.nestjs.com/techniques/performance#fastify-adapter) · [Fastify testing](https://fastify.dev/docs/latest/Guides/Testing/)

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

## Package policy

For Nest HTTP, use `@nestjs/common`, `@nestjs/core`, `@nestjs/platform-fastify`, and required framework peers such as `reflect-metadata` and `rxjs` for Nest 11. For Node.js HTTP without Nest, install `fastify` directly. Declare a direct `fastify` dependency in Nest only if the application imports it directly, and match the adapter-compatible version. Non-HTTP Node.js processes can use built-ins for their runtime work. For TypeScript, use `typescript` and `@types/node` as dev tooling, plus `@nestjs/cli` if the project builds through `nest build`. Check the installed or selected major's manifests. Fastify plugins are feature dependencies and follow the same approval rule.

Prefer built-in `fetch`, `AbortSignal.timeout()`, `node:crypto`, `Intl`, `node:test`, `node:assert/strict`, and `--env-file` on supported Node versions. Nest built-in `Parse*` pipes cover individual values. A small feature pipe can validate a DTO body. Use `ValidationPipe` from an existing project when its `class-validator` and `class-transformer` dependencies are already installed. Adding these packages requires explicit approval.

Official Nest packages such as `@nestjs/config`, `@nestjs/jwt`, `@nestjs/swagger`, `@nestjs/terminus`, `@nestjs/schedule`, and `@nestjs/testing` are feature choices. Verify their dependency requirements before choosing them. Database drivers, Redis clients, durable queue implementations, and provider SDKs require a supported existing integration or explicit approval for the required package. A compatible existing HTTP service can be called through `fetch` when its API meets the feature needs. Never hand-roll a Redis or database protocol, cryptographic token signing, or delivery guarantees to avoid a dependency.

See [Nest validation](https://docs.nestjs.com/techniques/validation), [Nest lifecycle](https://docs.nestjs.com/fundamentals/lifecycle-events), and [Node globals](https://nodejs.org/api/globals.html).

## Shared rate limiting

Login, reset/verification requests and sensitive mutations use shared limits where applicable. In a multi-instance backend, process-local Maps do not impose a global limit. Reuse the existing Redis limiter/package; add a short native script only when no suitable implementation exists. Bound key length/cardinality, trust forwarded IP only behind configured proxies, and use validated tenant/user/IP context. Hash sensitive normalized identifiers before using them as key segments; use HMAC with an existing secret when resistance to identifier guessing matters.

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
// RateLimitRepository, node-redis; key is built from controlled namespace/context.
const result = await this.redis.eval(FIXED_WINDOW_LIMIT, {
  keys: [key], arguments: [String(windowMs)],
}) as [number, number];
return { count: result[0], retryAfterMs: Math.max(0, result[1]) };

// RateLimitService/Guard; repository executes commands, guard gates transport.
// if (result.count > maxAttempts) throw the project's 429 with Retry-After.
```

Counter keys must be created only by this mechanism and not have their TTL removed. A fixed window permits boundary bursts; use the installed sliding/token-bucket limiter when the product requires smoother control, not a new custom algorithm. Redis outage must not be treated as zero attempts; choose explicit unavailable or a documented bounded policy, especially for security-sensitive routes. Test allowed, threshold, expiry and cross-instance/concurrent behavior against Redis when this implementation changes.

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
