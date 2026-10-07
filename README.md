# What Serverless Doesn't Handle For You

_Production finds the boundaries of serverless: connections that go stale, limits on request size and execution time, and background work and events that have to be processed reliably. This is where those boundaries are, and what to build at each one._

![Article cover - The title beside seven numbered boundaries, from dying connections to stateless realities](./assets/what-serverless-not-handle-full.avif)

Serverless is a genuine good deal for a small team. No servers to provision, automatic scaling, near-zero cost when idle, and fast deployment cycles. For the first few months, it usually works exactly as advertised.

Then production happens. Real users, real failures, real edge cases. And the problems that surface are not the ones the platform's getting-started guide mentions — because those guides are written to get you running, not to prepare you for what breaks at the boundaries.

This is about those boundaries. They follow a pattern: each one surfaces because serverless is doing exactly what it was designed to do — and that design collides with something the system needs.

---

## Part 1 — Your database connections keep dying

You build an app. It talks to a PostgreSQL database. Works fine in development, works fine in staging. In production, some users get intermittent 500 errors. Not consistently. Just sometimes. The database is healthy. Reloading works.

If you check your logs or Sentry dashboard, you see a trace that looks like this:

```text
Error: Connection terminated unexpectedly
    at Connection.parseE (/app/node_modules/pg/lib/connection.js:602:11)
    at Connection.parseMessage (/app/node_modules/pg/lib/connection.js:401:19)
    at Socket.<anonymous> (/app/node_modules/pg/lib/connection.js:121:22)
```

You have no idea what's wrong.

**What is actually happening**

Every time your app opens a connection to PostgreSQL, the operating system creates a TCP socket — a persistent, stateful channel between your process and the database server. A connection pool keeps several of these sockets open and reuses them across requests, which is faster than opening a new connection on every request.

Serverless introduces a problem that traditional servers never have. When a serverless function finishes handling a request, the platform does not shut the container down. It freezes it — suspends the entire process in memory — so the next request can start faster. This freeze also suspends the TCP sockets in the pool. From your application's perspective, the connections look perfectly healthy. From the database server's perspective, those connections have been completely silent for ten, fifteen, maybe thirty minutes.

Database servers and the network infrastructure between your function and your database — load balancers, NAT gateways, firewalls — all have idle timeout settings. When a connection sends no data for a certain period, they close it. They send a signal to the other side, but your frozen container is not listening. The socket on your side stays "open" in memory, but the server on the other end is gone. This is called a half-open connection.

When the next request resumes your container, the pool picks one of these dead sockets and hands it to the query. The query fails immediately. The user sees an error.

**Why the obvious fix doesn't work**

The instinct is to prevent the container from going idle. A cron job that pings the service every few minutes would keep the container warm and the connections alive.

This doesn't work for two structural reasons.

First, serverless platforms run many container instances simultaneously. A ping can only keep one instance alive. The platform routes each request to whichever container is available — it makes no promise about which instance a given request reaches. Your next real user request may land on a container that has been frozen for twenty minutes, with the same dead sockets.

Second, a typical health-check ping doesn't touch the database at all. The pool's connections are still idle. To actually keep connections alive, the ping would need to run a real database query on every call, across every instance, indefinitely — which means paying for constant compute and database traffic that produces nothing. At that point you are paying for always-on compute while getting none of what always-on infrastructure actually provides: instance affinity, predictable process state, connection ownership.

**The right approach**

The root cause is that a connection pool holds stateful TCP connections that become invalid when the process is suspended. The fix is to remove that state before suspension, and detect-and-recover when removal isn't possible.

Proactively: when the container is about to freeze, close all pool connections cleanly. The pool then has no stale sockets. Set an aggressive idle timeout on the pool — well below the database server's own idle timeout — so connections are evicted by your pool before the NAT gateway kills them silently.

```typescript
import { Pool } from '@neondatabase/serverless';
import { attachDatabasePool } from '@vercel/functions/oci';

const pool = new Pool({
  connectionString: process.env.DATABASE_URL,
  max: 1,                   // one connection per instance — serverless scales by spawning instances, not threads
                            // N concurrent instances × max connections must stay below your DB's connection limit
  idleTimeoutMillis: 10000, // evict after 10s idle — NAT gateways close at ~300s
  allowExitOnIdle: true,
});

// Registers a hook: drains pool connections before the container is frozen
attachDatabasePool(pool);
```

Reactively: lifecycle hooks don't catch every case — race conditions, provider edge cases, hooks that fire too late. A retry wrapper catches the specific error signatures of a dead socket and re-runs the query once with a fresh connection.

```typescript
async function withRetry<T>(fn: () => Promise<T>, attempts = 2): Promise<T> {
  for (let i = 0; i < attempts; i++) {
    try {
      return await fn();
    } catch (err) {
      const isStale = isStaleConnectionError(err);
      if (!isStale || i === attempts - 1) throw err;
      // pool will open a fresh connection on next acquire
    }
  }
  throw new Error('unreachable');
}

function isStaleConnectionError(err: unknown): boolean {
  if (!(err instanceof Error)) return false;
  return ['ECONNRESET', 'ECONNREFUSED', 'Connection terminated'].some(
    msg => err.message.includes(msg)
  );
}
```

For Redis: use an HTTP-based client rather than a TCP one. HTTP is stateless — each call opens a connection, completes, and closes. No pool, no frozen socket, no lifecycle management required.

```typescript
import { Redis } from '@upstash/redis';

// Reads UPSTASH_REDIS_REST_URL and UPSTASH_REDIS_REST_TOKEN from env
const redis = Redis.fromEnv();

// Every call is an independent HTTP request — no stale connection possible
await redis.set('session:123', JSON.stringify(sessionData), { ex: 3600 });
const session = await redis.get('session:123');
```

The broader principle: in serverless, any client that maintains persistent connections is a liability. Where you have a choice, choose stateless clients.

---

## Part 2 — Your platform rejects the request before your code runs

You add file uploads to your app. Users submit photos straight from a phone camera. Works in development. In production, uploads from some users fail silently or get rejected with no useful error. You check your logs. Your handler never ran.

**What is actually happening**

Serverless platforms sit behind a reverse proxy — an edge layer that validates and routes incoming requests before forwarding them to your function. This edge layer enforces hard limits on request body size to protect the shared infrastructure from memory exhaustion. On Vercel, the limit is 4.5MB. This is not a configurable default. It is enforced by the platform before your code is invoked.

A modern smartphone camera produces images between 8MB and 15MB. A form with two or three photos easily exceeds 4.5MB. Those requests are rejected at the edge. Your handler receives nothing, has no opportunity to respond, and the failure leaves no trace in your application logs.

**Why the obvious fixes don't work**

Compressing the images on the server: the request is rejected before your server code runs. There is no handler to do anything.

Increasing the limit in framework configuration: hard platform limits are not exposed as configuration options. The ceiling is the ceiling.

Breaking the file into multiple smaller requests and reassembling on the server: this does work around the per-request size limit, but it moves complexity into your API layer — you now write chunk assembly logic, handle out-of-order delivery, and manage incomplete uploads. You are still routing every binary byte through the serverless function, which is the problem. The limit changes from a single 12MB rejection to a ceiling of 4.5MB per chunk, not an elimination of the constraint.

All three approaches share the same mistake — they assume the file needs to flow through your API layer. It doesn't.

**The right approach**

Separate the coordination from the transfer.

Your API's job is lightweight: authenticate the request, validate the intent, issue a short-lived permission for the upload. The actual binary data should never pass through your serverless function at all. The client uploads directly to the storage provider using a presigned URL — a signed, time-limited URL that authorizes a specific upload to a specific location.

Server-side: generate the signed URL and return it. This response is a few dozen bytes. Nowhere near any size limit.

```typescript
// API route — generates a presigned URL. The file never touches this function.
export async function POST(req: Request) {
  const { filename, contentType } = await req.json();
  const key = `uploads/${crypto.randomUUID()}-${filename}`;

  const command = new PutObjectCommand({
    Bucket: process.env.STORAGE_BUCKET,
    Key: key,
    ContentType: contentType,
  });

  const signedUrl = await getSignedUrl(storageClient, command, { expiresIn: 300 });
  return Response.json({ url: signedUrl, key });
}
```

Client-side: compress first, then upload directly to storage using the signed URL. The Vercel API is not involved in the file transfer at all.

```typescript
async function uploadFile(file: File): Promise<string> {
  const compressed = await compressImage(file); // reduce before any network transfer

  // Step 1 — get a signed URL from your API (tiny JSON exchange)
  const { url, key } = await fetch('/api/presign', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ filename: file.name, contentType: 'image/jpeg' }),
  }).then(r => r.json());

  // Step 2 — upload directly to storage, bypassing Vercel's 4.5MB limit entirely
  await fetch(url, {
    method: 'PUT',
    body: compressed,
    headers: { 'Content-Type': 'image/jpeg' },
  });

  return key; // pass the key back to your API to associate with the record
}
```

One thing worth knowing: the AWS SDK v3 appends automatic checksum headers (`x-amz-checksum-crc32`) to presigned URLs by default. A browser `fetch()` PUT does not send those headers, causing the storage provider to reject the upload with a signature mismatch. Disable automatic checksums on the SDK client:

```typescript
const storageClient = new S3Client({
  region: 'auto',
  endpoint: `https://${process.env.R2_ACCOUNT_ID}.r2.cloudflarestorage.com`,
  credentials: {
    accessKeyId: process.env.R2_ACCESS_KEY!,
    secretAccessKey: process.env.R2_SECRET_KEY!,
  },
  requestChecksumCalculation: 'WHEN_REQUIRED',  // prevents browser upload rejection
  responseChecksumValidation: 'WHEN_REQUIRED',
});
```

The principle: when a platform enforces a hard size limit, the answer is not to compress or chunk through the limit. The answer is to route around the layer that enforces it.

---

## Part 3 — Your background work has a hard execution ceiling

After a user submits a form, things need to happen downstream: update a property management system, send a notification, write to a secondary data store. These don't need to block the user. So you defer them — fire them off after the response is sent, and move on.

Then you discover that some of that work never happened. No error. No log. Just missing data.

**What is actually happening**

Serverless platforms enforce a maximum execution duration on functions. On Vercel, the ceiling is 300 seconds on the Pro plan — and that limit applies to everything the function does, including any work triggered after the response is sent. This is the same kind of hard, platform-enforced limit as the 4.5MB body limit. You configure `maxDuration` in `vercel.json`, but you cannot exceed the plan ceiling.

```json
{
  "functions": {
    "src/app/api/**": {
      "maxDuration": 300
    }
  }
}
```

Vercel's `after()` runs a callback after the response is sent, inside the same container that handled the request. It looks like this:

```typescript
export async function POST(req: Request) {
  const data = await req.json();
  await saveSubmission(data); // the synchronous work

  after(async () => {
    // Runs after the response is sent.
    // Killed silently if: the container is recycled, or 300s total budget is reached.
    // No retry. No record. No error log.
    await notifyExternalSystem(data);
    await sendConfirmationEmail(data);
  });

  return Response.json({ ok: true });
}
```

That callback can be killed in two ways: the platform recycles the container before the callback finishes, or the 300-second budget is exhausted. When either happens, the callback stops with no retry and no record that it was running.

**Why the obvious fix doesn't work**

Increasing `maxDuration` buys more time within the ceiling. It does not remove the ceiling, and it does not make the callback durable. The container can still be recycled mid-execution. You are betting on timing, not solving reliability.

`after()` was designed for fast, low-stakes follow-up work — analytics pings, cache invalidations, non-critical notifications that can be safely dropped. Anything that requires reliable completion, meaningful compute time, or recovery on failure is asking `after()` to be something it was not designed to be.

**The right approach**

If the work cannot reliably run inside a serverless function within its constraints, it should not run there. Move it out to a dedicated worker and hand off work via a message queue.

---

## Part 4 — Dedicated workers: what they are and what options you have

The solution to Part 3 is not a smarter background callback. It is a different class of infrastructure: a dedicated worker process that receives work via a message queue and processes it without a time ceiling.

**The principle**

Separate accepting work from doing work. The serverless function receives the request, enqueues a job, and returns immediately — usually in under 200ms. The worker picks up the job and processes it with full compute resources and no platform-imposed deadline. The user doesn't wait. The worker doesn't care that the function that submitted the job was recycled minutes ago.

**Choosing a queue**

The one question that matters: *do you need the queue itself to be durable, or can your application compensate for queue failures?*

If you implement the outbox pattern from Part 5 — and you should — the queue is just a fast delivery path, not your source of truth. That changes the calculus significantly. A Redis-backed queue (BullMQ) is the right default for small teams: simple to operate, easy to add if Redis is already in the stack, and the outbox covers the durability gap. The weakness of Redis being in-memory first stops mattering once the database holds the authoritative record.

If you cannot use the outbox pattern — or you need the queue to be self-sufficient without a recovery process — a cloud-managed queue (SQS, Google Pub/Sub) provides at-least-once delivery backed by durable storage and dead-letter queues out of the box. The tradeoff is more operational surface and a cloud-vendor dependency.

HTTP-based queues (QStash, Inngest) are worth knowing about if you want push-based delivery without polling — the queue calls your webhook when a job is ready. Useful specifically when you cannot run a persistent worker process.

All three default to at-least-once delivery, which is why idempotency in the worker is non-optional regardless of which queue you choose. Exactly-once delivery is theoretically possible but practically impossible to guarantee at the queue protocol level alone — every system that claims it achieves it through idempotency at the application layer. Build for at-least-once and make your worker safe to run twice.

**Options for the worker**

*Cloud Run, App Runner, Container Apps* — managed container platforms that scale to zero when idle. The worker is a container image. The platform handles scaling and health checks. Cost model is similar to serverless for idle time, but with no hard execution limit.

*Fly.io and similar platforms (Railway, Render)* — persistent container platforms with less ops overhead than Kubernetes. The difference between them is mostly pricing and regional availability. Pick the one that fits your budget; the worker code is identical.

The worker code is straightforward. It polls the queue, processes each job, and acknowledges completion:

```typescript
import { Worker } from 'bullmq';

const worker = new Worker(
  'file-processing',
  async (job) => {
    const { fileKey } = job.data;

    // No 300-second ceiling — this runs until done
    const result = await processImage(fileKey);
    await writeResultToStorage(result.outputKey);
    await markJobComplete(job.id, result.outputKey);
  },
  {
    connection: { host: process.env.REDIS_HOST, port: 6379 },
    concurrency: 2, // process 2 jobs simultaneously per worker instance
  }
);

worker.on('failed', (job, err) => {
  logger.error({ jobId: job?.id, attempts: job?.attemptsMade }, `Job failed: ${err.message}`);
});
```

The serverless function that submits the job stays thin:

```typescript
import { Queue } from 'bullmq';

const queue = new Queue('file-processing', {
  connection: { host: process.env.REDIS_HOST, port: 6379 },
  defaultJobOptions: {
    attempts: 3,
    backoff: { type: 'exponential', delay: 2000 },
  },
});

export async function POST(req: Request) {
  const { fileKey } = await req.json();

  await queue.add('process-image', { fileKey, requestedAt: new Date().toISOString() });

  return Response.json({ status: 'queued' }); // returns in milliseconds
}
```

For a small team on serverless: a Cloud Run container running a BullMQ worker, scaled to zero when idle, is the pragmatic default. It fits the same cost model as serverless for idle time and removes the execution ceiling entirely.

---

## Part 5 — What if the message queue itself fails?

The dedicated worker architecture is correct. But there is still one failure mode hiding inside it.

Pushing a job to a queue is a network call. Network calls can fail — timeout, brief service interruption, connection error. If that call fails silently — no exception raised, just a timeout your error handling didn't catch — your application believes the job is queued. It is not. The work is gone, and nothing knows it's missing.

Before adopting a complex fix, make a deliberate product choice: *does it actually matter if this work is occasionally lost?* If you are enqueuing an analytics ping, a cache invalidation, or a non-critical notification, the outbox pattern is pure over-engineering. Just call the queue directly, log the occasional failure, and move on. But if you are processing a payment, triggering an inventory hold, or executing any core business transaction where failure means a broken state, leaving delivery to "best effort" is an architectural bug.

**The consistency problem**

When you write business data to your database and then enqueue a job, you are making two separate writes that both need to succeed. Databases support transactions — atomic, all-or-nothing operations. Queue systems do not participate in database transactions. You cannot commit a database row and an enqueue atomically. If the database commits and the enqueue fails, you have two systems in inconsistent state: the database knows the work was triggered, the queue has never heard of it.

**Why common fixes create new problems**

Retry the enqueue on failure: if the first enqueue actually succeeded and the acknowledgment was lost, retrying submits a duplicate. The worker processes the same job twice.

Wrap everything in a transaction: queue systems like Redis are not transactional participants. There is no rollback for an enqueue. The concept doesn't apply.

**The right approach: transactional outbox**

Accept that the queue is not durable enough to be the source of truth. Your database is. Use it.

Before attempting to enqueue anything, write a record of the intended work to your database — in the same transaction as the business operation that triggered it. The moment that row is committed, the work exists permanently and cannot disappear regardless of what the queue does. Then attempt the enqueue.

```typescript
// Step 1 — write the intent to the database. This is permanent.
const [event] = await db.insert(outboxEvents).values({
  id: crypto.randomUUID(),
  type: 'IMAGE_PROCESS_REQUESTED',
  payload: JSON.stringify({ fileKey }),
  status: 'pending',
  createdAt: new Date(),
}).returning();

// Step 2 — attempt to enqueue. Best effort. Failure is recoverable.
try {
  await queue.add('process-image', { eventId: event.id });
} catch (err) {
  // Not a crisis — the recovery scan will find this row and resubmit it
  logger.warn({ eventId: event.id }, 'Enqueue failed — recovery scan will resubmit');
}
```

A scheduled recovery process scans for rows that are older than a threshold and still marked as pending, then re-enqueues them. If the queue swallowed a job silently, the recovery process finds it. The data was never lost — it was in the database the whole time.

```typescript
// Recovery process — runs on a schedule (cron job, Cloud Run job, Vercel cron, etc.)
async function recoverPendingEvents() {
  const fiveMinutesAgo = new Date(Date.now() - 5 * 60 * 1000);

  const stale = await db
    .select()
    .from(outboxEvents)
    .where(
      and(
        eq(outboxEvents.status, 'pending'),
        lt(outboxEvents.createdAt, fiveMinutesAgo)
      )
    );

  if (stale.length > 0) {
    // Alert if the queue is consistently failing — a growing stale count means
    // enqueues are not succeeding and the recovery scan is doing all the work
    logger.warn({ count: stale.length }, 'Outbox recovery: resubmitting stale events');
  }

  for (const event of stale) {
    await queue.add(event.type, { eventId: event.id });
  }
}
```

The queue is an optimisation. Without it, the recovery scan delivers jobs in minutes. With it, jobs are delivered in seconds. Either way, nothing is lost.

**Idempotency is non-optional**

Re-enqueueing means the worker may process the same job more than once. The worker must produce the same result every time it sees the same event ID — whether it is the first time or the third.

For writes to your own database, use an upsert keyed on the event ID. A second write with the same ID is a safe no-op:

```typescript
// Worker processes an event — safe to run multiple times with the same eventId
async function processEvent(eventId: string) {
  const event = await db.query.outboxEvents.findFirst({
    where: eq(outboxEvents.id, eventId),
  });

  if (!event || event.status === 'completed') return; // already done

  const data = JSON.parse(event.payload);

  // Idempotent write — second call with same externalId is a no-op
  await db.insert(results)
    .values({ externalId: eventId, data: data.result })
    .onConflictDoUpdate({
      target: results.externalId,
      set: { data: sql`excluded.data` },
    });

  // Mark as completed so the recovery scan ignores it
  await db.update(outboxEvents)
    .set({ status: 'completed', completedAt: new Date() })
    .where(eq(outboxEvents.id, eventId));
}
```

For external API calls, send an idempotency key — a deterministic value derived from the event ID. A well-designed external API treats duplicate calls with the same key as no-ops after the first. But check the actual guarantee: many APIs only honor idempotency keys for a bounded window, typically 24–48 hours. A retry that arrives after that window is treated as a new call. Know what each external dependency actually promises.

---

## Part 6 — Your failed jobs are poisoning the queue

You successfully set up dedicated workers and a queue. Most jobs process smoothly. Then, a user uploads a corrupted file or submits a payload that triggers an unhandled edge case in your worker code. The worker throws an error. The queue retries the job. The job throws the same error. 

Your server logs start spinning with identical errors. Other users report their jobs are delayed. You check your worker, and it is entirely consumed by retrying that single broken job over and over again.

**What is actually happening**

This is the "poison pill" pattern. When a job fails due to a temporary network blip, retrying it is the correct response. But when a job fails due to a permanent issue — like a malformed payload, a corrupt binary, or a bug in your code — retrying it will never succeed. 

Without an explicit gate, the worker will consume that job, fail, wait for the backoff delay, and consume it again. If your queue has no retry limit, this loop runs indefinitely. If you have multiple worker instances, they can all become clogged with a handful of these poisoned jobs, completely starving legitimate jobs of CPU cycles and database connections.

**Why the obvious fixes don't work**

The first instinct is to wrap the worker's execution in a generic `try/catch` and swallow any error:

```typescript
try {
  await processJob(job);
} catch (err) {
  // Swallow the error so the queue thinks it succeeded and doesn't retry
  logger.warn({ jobId: job.id }, 'Job failed, swallowing error');
}
```

This prevents the poison pill loop, but it creates a silent failure. The job is marked as completed by the queue, the database state is never updated, and the user's request disappears. You have traded a resource clog for data loss, and you won't know something is broken until a customer contacts support.

The second instinct is to rely on passive monitoring — waiting for a developer to notice error spikes in Sentry or CloudWatch. By the time someone checks the dashboard, the system has been degraded for hours, database pools might be exhausted, and the queue backlog has ballooned.

**The right approach: Dead-letter queue (DLQ) with active monitoring**

Isolate the poison. If a job fails repeatedly, it should be removed from the main active channel and placed in a quarantine area — a Dead-Letter Queue (DLQ) or a dead jobs set. This keeps the active pipeline clear while preserving the failed job's data for analysis.

First: set a strict maximum retry limit on the queue (typically 3 to 5 attempts). Once a job exhausts all attempts, the queue library should automatically move it to a `failed` or `dead` state.

```typescript
// Enqueueing with strict attempt limits and exponential backoff
await queue.add('process-image', { fileKey }, {
  attempts: 5, // never retry indefinitely
  backoff: {
    type: 'exponential',
    delay: 2000, // wait 2s, 4s, 8s, 16s...
  }
});
```

Second: do not let failed jobs sit in the quarantine area silently. Implement active, automated monitoring of the DLQ depth. Run a scheduled task (such as a cron job or a background ticker) that checks the number of failed/dead jobs. If the count is greater than zero, raise an `ERROR` severity log immediately with a specific action instruction. This turns a passive storage dump into a loud, actionable alert.

```typescript
// Ticker-based monitor (runs every 5 minutes)
async function monitorDeadLetterQueue() {
  // BullMQ tracks permanently failed jobs in the 'failed' set
  const failedCount = await queue.getFailedCount();

  if (failedCount > 0) {
    // A non-empty DLQ is a system signal requiring active human intervention
    logger.error(
      { 
        count: failedCount, 
        action: 'Inspect failures using dashboard or run replay script' 
      },
      'Action Required: Jobs have accumulated in the Dead-Letter Queue'
    );
  }
}
```

Third: design for replayability. The data in the DLQ is precious — it represents exactly what the system failed to do. Once you diagnose the problem and deploy a hotfix, you need a way to move those quarantined jobs back into the active queue to complete the work without requiring the user to resubmit.

```typescript
// Admin utility to replay quarantined jobs
async function replayFailedJobs() {
  const failedJobs = await queue.getFailed();

  for (const job of failedJobs) {
    logger.info({ jobId: job.id }, 'Replaying quarantined job');
    // Move the job back to the active queue for reprocessing
    await job.retry();
  }
}
```

The principle: transient errors should be retried automatically; permanent errors must be quarantined, monitored actively, and replayed manually once resolved.

---

## Part 7 — Stateless scalability: what serverless actually is and isn't

Every problem in this article has the same root cause: serverless functions are stateless and ephemeral by design. That is not a bug — it is the entire point. Understanding what that means precisely is what separates a system that works in production from one that surprises you.

**What serverless is genuinely good at**

*Scale to zero* — when no requests are coming in, you pay nothing. For workloads with variable or unpredictable traffic — a webhook handler that fires occasionally, an API that is quiet at night — this is the most cost-efficient compute model available.

*Automatic horizontal scaling* — the platform handles spinning up additional instances when traffic increases. You do not write scaling logic, manage instance counts, or worry about capacity. A sudden traffic spike is absorbed automatically.

*No infrastructure management* — no OS patches, no server monitoring, no capacity planning. Engineers spend time on application code, not infrastructure operations.

*Fast deployment cycles* — deploying a new version is fast, rollbacks are simple, and preview environments are cheap. For a small team iterating quickly, this compounds into a real velocity advantage.

**What serverless is genuinely not good at**

*Stateful connections* — as Parts 1 through 3 explain, any client that maintains persistent connections becomes unreliable when the platform suspends and resumes containers. Serverless assumes statelessness. Persistent connections are state. They will always require explicit management.

*Long-running or sustained compute* — hard time limits (300 seconds on Vercel Pro), billing by the millisecond, and cold start penalties from large dependency trees make serverless a poor choice for image processing, ML inference, PDF generation, and anything else that needs seconds of sustained CPU. The cost model is wrong and the limits are hard.

*In-memory shared state* — two concurrent instances of a serverless function cannot share memory. Rate limiting counters, session data, job state — anything that needs to be shared across requests — must live in an external system. This is architecturally correct for scaling, but it means every shared-state operation involves a network call.

*Predictable latency* — cold starts introduce latency spikes. When containers are reclaimed after idle periods, the next request pays the initialization cost. For latency-sensitive workloads with tight SLAs, cold starts are an operational concern that requires active management (minimum warm instances, lightweight startup code).

**The tradeoff**

Serverless is the right default for a small team building a new system. The economics are real. The operational simplicity is real. Most of what a typical API does — read from a database, validate input, return JSON, write a row — fits perfectly within serverless constraints.

The problems in this article are not reasons to avoid serverless. They are the edges — places where the core assumptions of serverless (stateless, short-lived, ephemeral) collide with real-world requirements. They are predictable edges. Stateful connections will always need lifecycle management. Hard limits will always be there. Queue reliability will always require the outbox pattern.

The difference between a system that runs well and one that surprises you at midnight is knowing where those edges are before you reach them.

---

## Closing

Every managed platform makes tradeoffs on your behalf. Serverless trades infrastructure control for operational simplicity. That is a good trade for most small teams. What it does not do is eliminate the hard problems of distributed systems — connection reliability, payload size constraints, guaranteed delivery, durable state. It relocates them. They appear at different layers, in different forms, at different times in the growth of the system.

The patterns in this article — lifecycle-aware connection pools, presigned URL transfers, 300-second ceiling awareness, the dedicated worker model, the transactional outbox, idempotent processing, and dead-letter queue isolation — are not workarounds for serverless being broken. They are the correct designs for systems that run in ephemeral, stateless compute environments. They apply whether you are on Vercel, AWS Lambda, Google Cloud Run, or any other platform built around the same principles.

Learn where the platform ends and your responsibility begins. Build to that boundary deliberately. The platform handles the rest.

As Fred Brooks wrote in 1986 — and it has not stopped being true — "there is no silver bullet" in software engineering. Serverless is not the answer to everything. Neither is a dedicated server, a message queue, the outbox pattern, or a dead-letter queue. Each tool solves a specific class of problem and introduces its own tradeoffs. The work is choosing the right one for your constraints, avoiding the anti-patterns that come from reaching for the wrong one, and resisting the urge to over-engineer what does not yet need it. A system that is simple, boring, and maintainable beats a clever one every time.
