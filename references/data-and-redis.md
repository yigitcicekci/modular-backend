# Data and Redis

## Contents

- Queries, pagination, indexes
- Transaction ownership
- Redis roles and policies
- Redis configuration and provider
- Keys, TTLs, and command examples
- Redis-owned client configuration
- Consistency and verification

## Queries, pagination, indexes

Concrete repositories implement use-case operations, not a generic CRUD superclass. Select needed columns and bound results. Use supported bulk operations, nested reads, joins, or IN queries instead of per-item I/O. Parameterize raw SQL; never interpolate untrusted SQL text.

```ts
// BookRepository; isbns comes from a bounded request/page.
findMemberships(userId: string, isbns: string[]) {
  if (!isbns.length) return Promise.resolve([]);
  return this.prisma.libraryItem.findMany({
    where: { userId, isbn: { in: [...new Set(isbns)] } },
    select: { id: true, isbn: true, listType: true },
  });
}
```

```ts
// LibraryService: I/O before the loop; O(n+m) composition, O(m) lookup space.
const memberships = await this.books.findMemberships(userId, isbns);
const byIsbn = new Map(memberships.map((item) => [item.isbn, item]));
return results.map((book) => ({
  ...book, membership: byIsbn.get(book.isbn) ?? null,
}));
```

Promise.all over per-item queries is still per-item I/O. Use bounded concurrency only where a bulk API is unavailable. One ORM call is not proof of one SQL query; inspect generated queries for performance-sensitive paths. Do not assign O(1) to a database operation from its call syntax.

Aggregate in the database when the operation needs a total rather than full records; do not load an entire ledger/list into Node just to sum or count it. Keep exact decimal/money semantics and authorization scopes.

```ts
// WalletRepository: use the schema's decimal type; do not convert money to float.
sumAmounts(userId: string) {
  return this.prisma.walletTransaction.aggregate({
    where: { userId }, _sum: { amount: true },
  });
}
```

Null totals, currency boundaries and eligible transaction states remain domain decisions; use the actual ledger rules rather than treating this projection as a complete balance implementation.

Use cursor pagination for growing feeds; offset remains valid for bounded/page-number requirements. Use deterministic order and a matching unique cursor. This ID-sorted example is not chronological UUID pagination:

```ts
// UserRepository; limit validated once at boundary, e.g. 1..100.
async findPage(limit: number, cursor?: string) {
  const rows = await this.prisma.user.findMany({
    take: limit + 1,
    ...(cursor ? { cursor: { id: cursor }, skip: 1 } : {}),
    orderBy: { id: 'asc' },
    select: { id: true, username: true, createdAt: true },
  });
  return pageFromExtraRow(rows, limit);
}
```

```ts
// example: pagination
export function pageFromExtraRow<T extends { id: string }>(rows: T[], limit: number) {
  const hasMore = rows.length > limit;
  const items = hasMore ? rows.slice(0, limit) : rows;
  return { items, nextCursor: hasMore ? items[items.length - 1].id : null };
}
```

Helper preconditions: positive validated limit, at most limit+1 rows. Chronological lists need a stable timestamp/id order and corresponding cursor. Do not add count queries when the response needs no totals. Preserve current filters/cursor behavior on revisions.

Tie indexes to actual filters, joins, sorts, or constraints. Inspect primary/unique/composite coverage, selectivity, write cost, and representative plans. A PostgreSQL unique constraint already has an index: do not pair `username String @unique` with an equivalent `@@index([username])`. Composite-prefix overlap is a review candidate, not automatic proof an index is useless. For new schemas justify indexes from query shapes and label unmeasured assumptions. Never casually run costly/mutating EXPLAIN ANALYZE against production.

## Transaction ownership

Examples assume the PrismaClient/TransactionClient API. Match installed versions and generated imports; do not upgrade majors to fit examples.

```ts
// UserRepository; SessionRepository.create follows the same client rule.
create(
  data: CreateUserData,
  db: Prisma.TransactionClient | PrismaService = this.prisma,
) {
  return db.user.create({ data });
}
```

```ts
// AuthService: expensive preparation before opening the DB transaction.
const passwordHash = await this.passwords.hash(input.password);
const prepared = await this.tokens.prepare(userId, context);
const user = await this.prisma.$transaction(async (tx) => {
  const created = await this.users.create(
    { id: userId, email, username, passwordHash }, tx,
  );
  await this.sessions.create(prepared.session, tx);
  return created;
});
// Only return prepared tokens after commit.
```

Only participating methods need a transaction parameter. Every query in that operation uses the passed client, even through another service. Do not accidentally use a global client or open an independent nested transaction.

```ts
// UserRepository: self-contained storage operation; service decides eligibility.
disableAndRevoke(userId: string) {
  return this.prisma.$transaction([
    this.prisma.user.update({
      where: { id: userId }, data: { enabled: false },
    }),
    this.prisma.authSession.updateMany({
      where: { userId, revokedAt: null }, data: { revokedAt: new Date() },
    }),
  ]);
}
```

Do not hold transactions open for email, external HTTP, Redis I/O, or hashing. Conditional updates, row locks, or suitable isolation address races; default transaction isolation alone does not solve every race. Retry only known transient conflicts with a bound and safe repeatability. No blanket retry wrapper/Unit of Work framework.

## Redis roles and policies

Foundation work must ship a Redis-owned feature with a real write and read path. Use Redis for a suitable product responsibility such as published app config, live rankings, session state, or shared rate-limit counters. Choose hashes, sets, sorted sets, or strings around access patterns. Keep relational and financial data in the store that can enforce its invariants. A SQL response cache by itself does not fulfill the Redis data-store requirement.

For each key family record role, owner, population/update, TTL, miss, outage, and recovery in relevant config/code/project documentation:

| Role | Missing key | Outage | Population/recovery |
| --- | --- | --- | --- |
| Primary data | Domain absence; no SQL substitute | Explicit unavailable or documented safe behavior | Persistence and recovery according to loss tolerance |
| Prepared read model | Absence/not-ready; no request-time SQL fallback | Explicit unavailable | Initial load and write/event/job sync; readiness/rebuild |
| Disposable cache | Apply the feature's miss contract. Request-time SQL loading requires an explicitly agreed cache-aside policy. | Explicit unavailable for a required Redis read | TTL and appropriate update/invalidation |

Do not automatically route Redis failures to SQL or a per-process store. Existing fallback is observable behavior. Change it in a scoped remediation with tests. Preserve the difference between a true miss, required data that has not been published, and an outage.

## Redis configuration and provider

Keep the pieces discoverable in the project:

```text
src/config/redis.config.ts       # connection and runtime limits
src/config/redis-keys.config.ts  # namespaces and named key builders
src/config/redis-ttl.config.ts   # expirations with explicit units
src/libs/redis/redis.module.ts   # exports the shared provider
src/libs/redis/redis.service.ts  # client lifecycle and error boundary
```

Use the application's validated config loader to parse `REDIS_URL` (allow `redis:` and `rediss:`), namespace, connect deadline, and command deadline. Reject missing or invalid required config before serving traffic. Document configuration names in the example env file. Inject URLs and credentials at runtime, and omit them from logs and generic errors. Prefer a single client provider per process. Use a separate connection only for operations that need it, such as blocking reads or queue consumers. Wire Redis into local setup and production readiness.

```ts
// src/config/redis.config.ts; positiveInteger comes from the shared config validator.
export const REDIS_CONFIG = Symbol('REDIS_CONFIG');

export function readRedisConfig(env: NodeJS.ProcessEnv) {
  const url = env.REDIS_URL?.trim();
  let parsed: URL;
  try {
    parsed = new URL(url ?? '');
  } catch {
    throw new Error('REDIS_URL must be a valid Redis URL');
  }
  if (!['redis:', 'rediss:'].includes(parsed.protocol) || !parsed.hostname) {
    throw new Error('REDIS_URL must use redis: or rediss: and include a hostname');
  }
  const namespace = env.REDIS_NAMESPACE?.trim();
  if (!namespace || !/^[a-z0-9_-]{1,64}$/.test(namespace)) {
    throw new Error('REDIS_NAMESPACE must identify the app and environment');
  }
  const connectTimeoutMs = positiveInteger(
    env.REDIS_CONNECT_TIMEOUT_MS ?? '3000', 'REDIS_CONNECT_TIMEOUT_MS',
  );
  const commandTimeoutMs = positiveInteger(
    env.REDIS_COMMAND_TIMEOUT_MS ?? '1000', 'REDIS_COMMAND_TIMEOUT_MS',
  );
  if (connectTimeoutMs > 30_000 || commandTimeoutMs > 30_000) {
    throw new Error('Redis deadlines must be at most 30000 ms');
  }
  return {
    url: parsed.href, namespace, connectTimeoutMs, commandTimeoutMs,
  };
}

export type RedisConfig = ReturnType<typeof readRedisConfig>;
```

Register the validated object as a provider under `REDIS_CONFIG`, either in the project's config module or in `RedisModule`. Export `RedisService` from `RedisModule` and import it into modules that own Redis data. Tune the example bounds to the deployment's startup and request budgets. Example env values use a local URL and a unique non-secret namespace.

The following Nest provider fragment uses node-redis with `withCommandOptions` support. It assumes the project injects validated `REDIS_CONFIG` and maps `REDIS_UNAVAILABLE` to a typed HTTP 503. Adapt the lifecycle and deadline APIs for the installed client. Reuse ioredis if the project already runs it.

```ts
// src/libs/redis/redis.service.ts
import {
  Inject, Injectable, Logger, type OnApplicationShutdown, type OnModuleInit,
} from '@nestjs/common';
import { createClient } from 'redis';
import { AppException } from '../../common/exceptions/app.exception';
import { REDIS_CONFIG, type RedisConfig } from '../../config/redis.config';

@Injectable()
export class RedisService implements OnModuleInit, OnApplicationShutdown {
  private readonly logger = new Logger(RedisService.name);
  private readonly client: ReturnType<typeof createClient>;

  constructor(@Inject(REDIS_CONFIG) private readonly config: RedisConfig) {
    this.client = createClient({
      url: config.url,
      disableOfflineQueue: true,
      socket: {
        connectTimeout: config.connectTimeoutMs,
        reconnectStrategy: (attempt) => attempt < 3 ? 100 * (attempt + 1) : false,
      },
    });
    this.client.on('error', () => this.logger.warn('Redis connection error'));
  }

  async onModuleInit() {
    try {
      await this.client.connect();
      await this.run((redis) => redis.ping());
    } catch {
      if (this.client.isOpen) this.client.destroy();
      throw new AppException('REDIS_UNAVAILABLE');
    }
  }

  async run<T>(operation: (redis: ReturnType<typeof createClient>) => Promise<T>) {
    if (!this.client.isReady) throw new AppException('REDIS_UNAVAILABLE');
    const controller = new AbortController();
    let timer: ReturnType<typeof setTimeout> | undefined;
    try {
      const deadline = new Promise<never>((_, reject) => {
        timer = setTimeout(() => {
          controller.abort();
          reject(new AppException('REDIS_UNAVAILABLE'));
        }, this.config.commandTimeoutMs);
      });
      const redis = this.client.withCommandOptions({ abortSignal: controller.signal });
      return await Promise.race([operation(redis), deadline]);
    } catch {
      this.logger.warn('Redis command could not complete');
      throw new AppException('REDIS_UNAVAILABLE');
    } finally {
      clearTimeout(timer);
    }
  }

  onApplicationShutdown() {
    if (this.client.isOpen) this.client.destroy();
  }
}
```

Keep the callback restricted to one Redis command or a single batch. Handle domain decisions and decoding outside this boundary. Use established classification and telemetry for malformed commands, permission failures, and dropped connections. Bound startup retries and choose runtime reconnect behavior for the deployment model. The deadline bounds the caller's wait. A command already sent may continue on the server. Use appropriate queue bounds and avoid blind retries of non-idempotent writes. Keep blocking reads on a separate connection. Drain workers and HTTP requests before closing their shared client. [Redis production client guidance](https://redis.io/docs/latest/develop/clients/nodejs/produsage/)

## Keys, TTLs, and command examples

Reuse existing key builders. A new project can use typed functions; no template parser needed:

```ts
// src/config/redis-keys.config.ts, relative keys
export const REDIS_KEYS = {
  publishedClientConfig: 'client-config:published',
  session: (id: string) => `auth:session:${id}`,
  userSessions: (id: string) => `auth:user-sessions:${id}`,
  product: (id: string) => `catalog:product:${id}`,
  tournamentRanks: (id: string) => `tournament:${id}:ranks`,
  libraryCards: 'library:cards',
} as const;
```

```ts
// src/config/redis-ttl.config.ts
export const REDIS_TTL = {
  productSeconds: 300,
  loginWindowSeconds: 60,
} as const;
```

Values illustrate a policy. Prefix keys through one builder with the injected application/environment namespace. Use validated IDs or encode arbitrary segments to avoid collisions. Keep tokens, passwords and raw personal identifiers out of keys. Apply TTL to the specific key families that expire. Shared Redis infrastructure owns the connection. Repositories own domain commands and decide which keys to read or write.

```ts
// TournamentRedisRepository; node-redis method names.
async rank(tournamentId: string, userId: string) {
  const value = await this.redis.zRevRank(
    REDIS_KEYS.tournamentRanks(tournamentId), userId,
  );
  return value === null ? null : value + 1;
}

// LibraryRedisRepository: one batch, no invalid empty HMGET.
getCards(isbns: string[]) {
  return isbns.length
    ? this.redis.hmGet(REDIS_KEYS.libraryCards, isbns)
    : Promise.resolve([]);
}
```

Adapt to the project's existing Redis client or approved service interface. If a new client is the only supported option, state that requirement before adding a package. Pipelining reduces round trips. Use native atomic commands, MULTI/EXEC, or bounded Lua when an operation must also be atomic. Test the actual invariant.

Use the following only when the product explicitly selects cache-aside for a disposable response cache. Never use it for Redis-owned records or prepared projections. The adapter sets key and TTL from config. Both GET and SET errors propagate.

```ts
// example: cache-aside
export async function readCached<T>(
  cache: { get(): Promise<T | null>; set(value: T): Promise<void> },
  load: () => Promise<T>,
): Promise<T> {
  const cached = await cache.get();
  if (cached !== null) return cached;
  const value = await load();
  await cache.set(value);
  return value;
}
```

This example does not coalesce concurrent misses or guarantee transactional freshness. Use for suitable bounded reads; hot keys may need existing coalescing/bounded synchronization. If returning a loaded value despite SET failure is appropriate, implement/test that distinct policy explicitly. Test null separately from valid false/zero/empty-string values. Do not add this generic helper if existing module code already covers the need.

## Redis-owned client configuration

Use this pattern when the product needs a minimum client version, feature flags, or another published runtime policy. Keep infrastructure env configuration in the normal config loader. Store the published business configuration in Redis and expose only its public fields. The example keeps the real minimum supported build as a single Redis key with no TTL.

```ts
// ClientConfigRepository; key() adds the validated namespace.
async findMinimumBuild(): Promise<number | null> {
  const value = await this.redis.run((redis) =>
    redis.get(this.key(REDIS_KEYS.publishedClientConfig)),
  );
  if (value === null) return null;
  const minimumBuild = Number(value);
  if (!Number.isSafeInteger(minimumBuild) || minimumBuild < 1) {
    throw new AppException('CLIENT_CONFIG_NOT_READY');
  }
  return minimumBuild;
}

async publishMinimumBuild(minimumBuild: number): Promise<void> {
  await this.redis.run((redis) =>
    redis.set(this.key(REDIS_KEYS.publishedClientConfig), String(minimumBuild)),
  );
}
```

```ts
// ClientConfigService; the public controller delegates here.
async getPublishedConfig() {
  const minimumBuild = await this.repository.findMinimumBuild();
  if (minimumBuild === null) throw new AppException('CLIENT_CONFIG_NOT_READY');
  return { minimumBuild };
}

// Called by the authorized publisher with a validated PublishClientConfigDto.
async publish(input: PublishClientConfigDto) {
  await this.repository.publishMinimumBuild(input.minimumBuild);
  return input;
}
```

Wire the publisher into the actual authorized write or provisioning path. Publish initial policy before serving routes that require it. The test fixture seeds Redis through that same repository, asserts the public API read, and verifies an absent key gives the defined not-ready response. A Redis outage follows `REDIS_UNAVAILABLE`. There is no SQL loader for either case. A prepared projection can be refreshed or rebuilt by a controlled writer. For persistent Redis-owned records, choose persistence, recovery and a memory policy that protects them from eviction. Check restore with disposable infrastructure before relying on it in production.

## Consistency and verification

- Primary records do not get arbitrary cache TTLs. Define loss/staleness tolerance, eviction, persistence, backup/restore and rebuild. Persistence is not a zero-loss promise. Separate Redis storage roles when eviction/durability needs conflict; logical DB numbers do not isolate eviction policy.
- SQL and Redis writes do not share a DB transaction. Define commit order, invalidation/update timing, failure and recovery. Never report session revocation as effective while accepting stale authorized cache state without a stated bounded-revocation contract.
- Reuse existing jobs/events/CDC for prepared data. If reliable propagation is required, add the minimum durable mechanism needed, such as an outbox; do not rely on a crash-lost background promise. Do not introduce outboxes for every cache.
- Use locks only for a concrete invariant; define owner, expiry, ownership-checked release and recovery. Redis Cluster multi-key operations need co-located keys and compatible scripting. Keep Lua bounded; do not scan unbounded collections atomically.
- Revisions update all key readers/writers/jobs with any required compatibility transition. Never flush the whole Redis instance or discard active sessions as cleanup.
- Verify miss/outage/empty values, update/invalidation, rollback, relevant races and query count. Do not claim N+1 removed solely from ORM call count.

Official sources: [Prisma queries](https://www.prisma.io/docs/orm/v6/prisma-client/queries/query-optimization-performance), [transactions](https://www.prisma.io/docs/orm/v6/prisma-client/queries/transactions), [PostgreSQL unique indexes](https://www.postgresql.org/docs/current/indexes-unique.html), [Redis cache-aside](https://redis.io/docs/latest/develop/use-cases/cache-aside/), [prefetch cache](https://redis.io/docs/latest/develop/use-cases/prefetch-cache/), [persistence](https://redis.io/docs/latest/operate/oss_and_stack/management/persistence/), [pipelining/transactions](https://redis.io/docs/latest/develop/clients/nodejs/transpipe/).
