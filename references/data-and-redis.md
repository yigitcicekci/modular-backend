# Data and Redis

## Contents

- Queries, pagination, indexes
- Transaction ownership
- Redis roles and policies
- Keys, TTLs, and command examples
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

Use Redis for workload-appropriate sessions, counters, presence, rankings, prepared catalogs/config, queues, or cached reads. Choose hashes, sets, sorted sets, or strings by access pattern; avoid rewriting serialized collections for individual updates. Do not move relational/financial truth merely to avoid SQL.

For each key family record role, owner, population/update, TTL, miss, outage, and recovery in relevant config/code/project documentation:

| Role | Missing key | Outage | Population/recovery |
| --- | --- | --- | --- |
| Primary data | Domain absence; no SQL substitute | Explicit unavailable or documented safe behavior | Persistence and recovery according to loss tolerance |
| Prepared read model | Absence/not-ready; no request-time SQL fallback | Explicit unavailable | Initial load and write/event/job sync; readiness/rebuild |
| Disposable cache | Cache-aside may load from owning repository | No automatic catch-to-DB fallback; choose explicitly from capacity/freshness needs | TTL and appropriate update/invalidation |

Choose the method per role: do not universally ban cache-aside misses or silently add DB fallback on connection errors. Existing fallback is observable behavior; change only within task scope. Never turn storage failure into genuine empty/null/zero or implicit authorization success.

## Keys, TTLs, and command examples

Reuse existing key builders. A new project can use typed functions; no template parser needed:

```ts
export const REDIS_KEYS = {
  session: (id: string) => `auth:session:${id}`,
  userSessions: (id: string) => `auth:user-sessions:${id}`,
  product: (id: string) => `catalog:product:${id}`,
  tournamentRanks: (id: string) => `tournament:${id}:ranks`,
  libraryCards: 'library:cards',
} as const;
export const REDIS_TTL = {
  productSeconds: 300,
  loginWindowSeconds: 60,
} as const;
```

Values illustrate policy, not universal lifetimes. Use validated IDs/encoded arbitrary segments; namespace shared deployments/tenants. No bearer tokens/passwords/raw personal identifiers in keys. Domain repositories own Redis commands; the shared provider owns connection lifecycle/serialization, not wrappers for Redis's entire API.

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

The helper below assumes a repository adapter with typed values/null and an explicitly selected cache-aside policy. The adapter sets key/TTL from config. It intentionally propagates GET and SET failures; no SQL fallback on outage.

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

Prepared/primary Redis reads receive no SQL loader:

```ts
// AppConfigService calls AppConfigRedisRepository.
const config = await this.repository.findPublished();
if (config === null) throw new AppException('APP_CONFIG_NOT_READY');
return config;
```

## Consistency and verification

- Primary records do not get arbitrary cache TTLs. Define loss/staleness tolerance, eviction, persistence, backup/restore and rebuild. Persistence is not a zero-loss promise. Separate Redis storage roles when eviction/durability needs conflict; logical DB numbers do not isolate eviction policy.
- SQL and Redis writes do not share a DB transaction. Define commit order, invalidation/update timing, failure and recovery. Never report session revocation as effective while accepting stale authorized cache state without a stated bounded-revocation contract.
- Reuse existing jobs/events/CDC for prepared data. If reliable propagation is required, add the minimum durable mechanism needed, such as an outbox; do not rely on a crash-lost background promise. Do not introduce outboxes for every cache.
- Use locks only for a concrete invariant; define owner, expiry, ownership-checked release and recovery. Redis Cluster multi-key operations need co-located keys and compatible scripting. Keep Lua bounded; do not scan unbounded collections atomically.
- Revisions update all key readers/writers/jobs with any required compatibility transition. Never flush the whole Redis instance or discard active sessions as cleanup.
- Verify miss/outage/empty values, update/invalidation, rollback, relevant races and query count. Do not claim N+1 removed solely from ORM call count.

Official sources: [Prisma queries](https://www.prisma.io/docs/orm/v6/prisma-client/queries/query-optimization-performance), [transactions](https://www.prisma.io/docs/orm/v6/prisma-client/queries/transactions), [PostgreSQL unique indexes](https://www.postgresql.org/docs/current/indexes-unique.html), [Redis cache-aside](https://redis.io/docs/latest/develop/use-cases/cache-aside/), [prefetch cache](https://redis.io/docs/latest/develop/use-cases/prefetch-cache/), [persistence](https://redis.io/docs/latest/operate/oss_and_stack/management/persistence/), [pipelining/transactions](https://redis.io/docs/latest/develop/clients/nodejs/transpipe/).
