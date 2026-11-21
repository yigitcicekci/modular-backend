# Background work, idempotency, and resource budgets

## Contents

- API/worker separation
- Job manager and pipelines
- Queue delivery and retry semantics
- Webhook idempotency
- Scheduling and bounded batches
- Pooling, metrics, shutdown, and checks

## API/worker separation

Move slow notification/fanout/export/CPU work out of request handling when latency, reliability, or scale requires it. Do not add a queue to simple synchronous work. Reuse the installed queue; BullMQ below is an example, not a required dependency.

Keep API producers and background consumers in separate module graphs/entrypoints when they need independent scaling. API replicas must not each register cron schedules or start consumers unintentionally. One immutable image can serve API and worker roles with different validated entry commands; sharing an image does not mean loading the same full module graph.

```ts
// worker.ts; WorkerModule imports consumers and their dependencies, no HTTP server.
async function bootstrapWorker() {
  const app = await NestFactory.createApplicationContext(WorkerModule);
  app.enableShutdownHooks();
}
bootstrapWorker().catch((error) => {
  logger.error({ event: 'WORKER_START_FAILED', error: safeError(error) });
  process.exitCode = 1;
});
```

Use the project's safe logger and fail-fast lifecycle; close partially initialized resources on bootstrap failure. Do not keep a half-initialized process running merely because its port opened. In containers, start the actual Node process with signals reaching it; do not add PM2 inside every container without a concrete process-management need.

## Job manager and pipelines

Use `@nestjs/schedule` for real periodic work, with `ScheduleModule.forRoot()` in the scheduler graph and a typed `JOBS_ENABLED` flag defaulting to false until intentionally enabled. Use an API graph without the scheduler and consumers. Long or retryable delivery can use an existing queue, such as BullMQ for a Redis-based worker, when a direct scheduled domain call is insufficient. Register the producer and processor against the same queue name and configuration. Avoid a queue or generic pipeline framework for a single task.

When a workflow has multiple dependent steps, a `JobManagerService` maps triggers to stable job IDs and calls typed domain steps in order. Keep each step in its owning service. Add a `JobParameterRepository` only for operational settings that must change without a deploy. Upsert parameter rows by job ID, validate bounded values when read, and track per-job enablement separately from the process flag. For audit/history needs, persist run ID, trigger, start/end timestamps, and per-step counts or status. Use a terminal `failed` status if a required step throws. Do not log raw payloads or return success after partial failure.

```ts
// src/workers/jobs/job-manager.service.ts
import { Injectable } from '@nestjs/common';
import { ConfigService } from '@nestjs/config';
import { Cron } from '@nestjs/schedule';
import { MaintenanceService } from '../../modules/maintenance/maintenance.service';

@Injectable()
export class JobManagerService {
  constructor(
    private readonly config: ConfigService<{ JOBS_ENABLED: boolean }, true>,
    private readonly maintenance: MaintenanceService,
  ) {}

  @Cron('15 4 * * *', { name: 'daily-maintenance', timeZone: 'UTC', waitForCompletion: true })
  async dailyMaintenance() {
    if (!this.config.get('JOBS_ENABLED', { infer: true })) return { status: 'disabled' } as const;
    const now = new Date();
    const runKey = `daily-maintenance:${now.toISOString().slice(0, 10)}`;
    const archived = await this.maintenance.archiveEligible(runKey, now);
    const deleted = await this.maintenance.pruneExpired(runKey, now);
    return { status: 'completed', archived, deleted } as const;
  }
}
```

Configure the boolean through startup validation, so the string `false` cannot enable execution. Register the provider in the worker module, with the config provider and owning domain module imported. The service methods use idempotent transitions or durable per-step checkpoints based on the operation. A UTC date key identifies the scheduled period, while a separate attempt ID identifies an individual run. If catch-up is required, pass the target scheduled period explicitly instead of assuming today's date. A local `waitForCompletion` skips overlapping ticks in that process. If replicas or deployment overlap can trigger the same work, add an atomic claim and idempotency in storage or use queue-native scheduling. [Nest task scheduling](https://docs.nestjs.com/techniques/task-scheduling)

Manual work uses the same domain entry point and is auditable. Expose a CLI or an explicitly authorized operator route only when needed. Return HTTP 202 after successful durable enqueue, or return the outcome if execution is synchronous. A mobile-header exemption alone is insufficient for operator access.

When operators need stored parameters, the repository can provision a job with an atomic upsert. Keep the typed defaults small and validate overrides before the runner uses them.

```ts
// JobParameterRepository, Prisma. JobParameter.id is the unique job ID.
getOrCreate(id: string, defaults: { batchSize: number; maxBatches: number }) {
  return this.prisma.jobParameter.upsert({
    where: { id },
    create: { id, disabled: true, parameters: defaults },
    update: {},
    select: { disabled: true, parameters: true },
  });
}
```

## Queue delivery and retry semantics

Queue consumers call the domain service; they do not duplicate its business rules or issue ad hoc SQL. Queue state is not proof that a business side effect happened exactly once. Assume at-least-once delivery with duplicates, retries and worker crashes. Use stable business operation IDs, bounded retries/backoff, provider limits, and meaningful failed-job retention/recovery.

```ts
// NotificationProducerService, with shared queue policy from config.
await this.queue.add('send-notification', { deliveryId }, {
  ...QUEUE_POLICY.notification,
  jobId: `notification-${deliveryId}`,
});
```

deliveryId is a validated stable UUID/operation ID, not Date.now or an attempt ID. Use a supported BullMQ custom ID format. Job IDs deduplicate only while the old job record exists; auto-removal/retention must match the intended retry horizon. Do not claim permanent idempotency from jobId alone.

```ts
@Processor(NOTIFICATION_QUEUE)
export class NotificationWorker extends WorkerHost {
  constructor(private readonly delivery: NotificationDeliveryService) { super(); }

  async process(job: Job<{ deliveryId: string }>) {
    if (job.name !== 'send-notification') {
      throw new Error('Unsupported notification job');
    }
    await this.delivery.deliver(job.data.deliveryId);
  }
}
```

Set concurrency in validated worker configuration. Bound it by DB/Redis pools, provider rate limits and workload; do not hardcode a production concurrency copied from an example. Validate payloads at external queue trust boundaries and version payloads if old jobs must survive a rollout. Configure transient versus permanent failures through the queue's supported mechanism; do not retry malformed/unauthorized work endlessly. Do not return success for an unknown job.

An external send can succeed before a worker crashes and marks the job complete. Use the provider's idempotency capability or an explicit acceptable duplicate contract; a DB sent flag alone cannot guarantee exactly-once external delivery. Transactional outbox solves durable DB-to-queue publication, not exactly-once provider execution. Add it only when the product needs reliable propagation and no existing mechanism covers it.

## Webhook idempotency

Authenticate the provider signature using the required raw body and timestamp/replay policy before trusting event identity. Persist the provider's stable event ID with a uniqueness constraint. Never invent dedupe IDs from receipt time, raw purchase secrets, or an arbitrary client field. An idempotency lookup failure is unavailable, not “not processed.”

For a PostgreSQL/Prisma handler that mutates only the same database, insert the receipt and apply the mutation in one transaction:

```prisma
model WebhookReceipt {
  id         String   @id @default(uuid())
  provider   String
  eventId    String
  receivedAt DateTime @default(now())
  @@unique([provider, eventId])
}
```

```ts
// WebhookReceiptRepository; skipDuplicates requires a supporting connector.
async claim(provider: string, eventId: string, tx: Prisma.TransactionClient) {
  const result = await tx.webhookReceipt.createMany({
    data: [{ provider, eventId }], skipDuplicates: true,
  });
  return result.count === 1;
}
```

```ts
// BillingWebhookService; event has already been verified and normalized.
return this.prisma.$transaction(async (tx) => {
  const claimed = await this.receipts.claim(event.provider, event.id, tx);
  if (!claimed) return { duplicate: true };
  await this.subscriptions.applyVerifiedEvent(event, tx);
  return { duplicate: false };
});
```

All participating repositories use tx. A failed mutation rolls back the receipt so retry can process it. If the effect is external, write a durable outbox record inside the transaction and deliver separately; do not place external HTTP inside it. Duplicate delivery and out-of-order delivery are different: enforce the domain's provider version/state-transition rules so an older event cannot regress current state. Set receipt retention from the provider replay horizon and business consequence, not arbitrary cleanup.

## Scheduling and bounded batches

Use a single scheduler role, queue-native scheduler, or a suitable distributed claim when multiple instances could trigger the same task. State the intended timezone explicitly; do not inherit the developer machine's timezone. Protect against overlap and design re-entry/retry safely. Do not build a generic pipeline DSL for a few sequential service calls.

```ts
// BatchService; repository implements stable cursor order and bounded reads.
let cursor = startingCursor;
for (let page = 0; page < JOB_POLICY.maxBatches; page++) {
  const batch = await this.repository.findPendingAfter(cursor, JOB_POLICY.batchSize);
  if (!batch.length) return { exhausted: true };
  await this.processor.processBatch(batch);
  cursor = batch[batch.length - 1].id;
}
return { exhausted: false, nextCursor: cursor };
```

Cursor and pending-state mutations must not skip records; checkpoint/retry behavior belongs to the actual operation. When a run reaches its budget, save/schedule the continuation if necessary for timely completion. Load recipient tokens/languages in batches, use Map lookups, and apply bounded provider batches rather than per-user SQL or unbounded Promise.all. A batch-size constant belongs in shared job config when reused. Keep the operation retryable after a partial batch; do not mark the entire run successful when a required step failed.

For a short Redis coordination lease, acquire atomically with value and expiry in one SET. Do not split SETNX and EXPIRE. Releasing requires ownership checking, and a lease alone is insufficient for critical work that could outlive its expiry; use fencing/conditional writes or queue ownership where required.

```ts
// CoordinationRepository, node-redis. owner is a unique per-acquisition value.
const acquired = await this.redis.set(key, owner, { NX: true, PX: leaseMs });
return acquired === 'OK';
```

## Pooling, metrics, shutdown, and checks

Budget DB connections across maximum API replicas, workers, schedulers, deployment overlap, migrations and operational reserve. A per-process pool maximum multiplied by replicas can exceed the DB limit. Keep one managed pool/client per process where the library expects it, validate settings, and close owned resources. Do not instantiate clients/pools per request. CPU-heavy work must not block the API event loop or prevent a queue worker renewing its lock; use a suitable worker thread/process when required.

Measure request duration/error rate, DB pool wait/saturation, Redis failures/hit-miss where meaningful, and queue lag/failures/stalls. Metric labels must have bounded cardinality: route templates/job types/status, not user IDs, raw URLs, tokens or arbitrary event IDs. Use the installed telemetry stack; do not require a monitoring vendor.

Shutdown: stop accepting new work, drain in-flight requests/jobs within the platform grace period, then close owned DB/Redis/queue clients. Queue crashes may still redeliver; graceful shutdown is not an idempotency guarantee. For a plain BullMQ worker owned directly by a lifecycle provider:

```ts
async onApplicationShutdown() {
  await this.worker.close();
}
```

Check an existing Nest queue integration's shutdown handling before registering a duplicate closer. Align worker drain time with orchestrator termination policy; define stuck-job recovery and shutdown deadline handling.

Verify duplicated/out-of-order webhooks, rollback-after-claim, worker retry/crash, unknown-job failure, concurrent scheduler claims, lease expiry, bounded batches and pool behavior as relevant. Use real DB/Redis integration checks for concurrency claims. Do not infer exactly-once behavior from mock tests.

Sources: [BullMQ idempotent jobs](https://docs.bullmq.io/patterns/idempotent-jobs), [job retention](https://docs.bullmq.io/guide/queues/auto-removal-of-jobs), [graceful shutdown](https://docs.bullmq.io/guide/workers/graceful-shutdown), [stalled jobs](https://docs.bullmq.io/guide/jobs/stalled), [Redis SET](https://redis.io/docs/latest/commands/set/).
