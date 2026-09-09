# Scheduled data cleanup: draining a rate-limited worker queue of stale uploads

Use a durable job table, one rate limiter shared by the whole worker pool, and delete calls that are safe to repeat. A scheduled data cleanup that removes stale uploads every night is a queue-draining problem wearing a cron costume, and the part that decides whether you sleep through the night is what happens on attempt number two. The nightly trigger should do exactly one cheap thing: enqueue work. Everything after that belongs to workers that drain the queue at whatever rate the object store will accept.

That split matters far more than which scheduling library you pick.

Make it concrete with a developer-tools SaaS running in the EU. Uploads carry a 30-day retention promise, the deletes have to happen against a bucket in the same region, and the store starts answering 429 somewhere north of ten delete calls per second. Two hundred thousand objects come due tonight. One loop, one process, one long transaction — that design fails in a boring way at 60% through, and the next night it starts from the top and re-walks everything it already handled.

## The two shapes of a nightly sweep

Picture the old shape as a straight line: cron fires, a query selects every expired row, the process walks the list, and each iteration calls delete. There's no memory between iterations. Kill the container at 02:40 and the only record of progress is whatever the store already deleted, which your database doesn't know about. Retry means replay from zero.

Now picture the shape you want as a fork. The scheduler writes rows into a `upload_cleanup_jobs` table and exits in under a second. A pool of workers — eight of them, say — each claim one job at a time, hold a short lease on it, spend a token from a shared bucket, issue the delete, then mark the row done. The queue is the memory. Progress survives a deploy, a crash, or a rolling restart, because the unit of work is a row with a state column, not a position in a for-loop.

Both shapes delete the same objects. Only one of them can be interrupted.

## How do you delete stale uploads nightly without stalling the whole worker queue?

Two mechanisms, and they solve different problems. The limiter protects the store from your pool; leases plus idempotency protect your data from your own retries.

Rate limiting belongs in front of the delete call, not around the batch. A token bucket sized to the whole pool — ten tokens per second, burst of twenty — means adding workers changes throughput not at all, which is the point. Per-worker limiters are the classic mistake here: eight workers each politely capped at ten per second still hit the bucket at eighty.

Idempotency is the other half, and it's mostly a matter of deciding what "already gone" means. A delete that returns 404 succeeded — the object is absent, which is the state you asked for. Treat that as done, not as an error to retry. Key each job by object id so a duplicate insert collides instead of queuing a second delete, and make the terminal write a state transition rather than a counter increment.

| Failure mode | What it looks like at 03:00 | What actually fixes it |
| --- | --- | --- |
| Worker killed mid-delete | Job stuck in `running` forever | A lease with an expiry, not a boolean flag |
| Store answers 429 | Error burst right after the sweep starts | Shared token bucket, plus honoring `Retry-After` |
| Same job claimed twice | Duplicate delete, noisy alerting | `FOR UPDATE SKIP LOCKED`, and 404 counted as success |
| Batch bigger than the window | Cleanup still running at 09:00 | A hard deadline; leftovers roll into tomorrow |

The lease deserves one more sentence, because it's the piece people skip. A boolean `claimed` column has no recovery story — the sweeper that crashed owns that row until a human intervenes. A `lease_until` timestamp lets the next claim query pick the row back up automatically, which is the difference between a stuck queue and a slow one.

## The drain loop, in code

Claiming is where PostgreSQL does the heavy lifting. `FOR UPDATE SKIP LOCKED` lets each worker grab rows nobody else holds, with no coordinator and no distributed lock.

```ts
import { setTimeout as sleep } from "node:timers/promises";
import { Pool } from "pg";

const db = new Pool({ connectionString: process.env.DATABASE_URL });

// Cap the whole pool, not each worker.
const RATE_PER_SEC = 10;
const BURST = 20;
let tokens = BURST;
let lastRefill = Date.now();

async function takeToken(): Promise<void> {
  for (;;) {
    const now = Date.now();
    tokens = Math.min(BURST, tokens + ((now - lastRefill) / 1000) * RATE_PER_SEC);
    lastRefill = now;
    if (tokens >= 1) {
      tokens -= 1;
      return;
    }
    await sleep(Math.ceil(((1 - tokens) / RATE_PER_SEC) * 1000));
  }
}

type Job = { id: string; bucket: string; key: string; attempts: number };

async function claim(): Promise<Job | undefined> {
  const { rows } = await db.query<Job>(
    `UPDATE upload_cleanup_jobs j
        SET state = 'running',
            lease_until = now() + interval '2 minutes',
            attempts = attempts + 1
       FROM (
         SELECT id FROM upload_cleanup_jobs
          WHERE run_after <= now()
            AND (state = 'pending' OR (state = 'running' AND lease_until < now()))
          ORDER BY run_after
          LIMIT 1
          FOR UPDATE SKIP LOCKED
       ) pick
      WHERE j.id = pick.id
      RETURNING j.id, j.bucket, j.key, j.attempts`,
  );
  return rows[0];
}
```

The delete side is where idempotency earns its keep. Your storage client is a stand-in here — any S3-compatible SDK, or plain HTTP against the bucket endpoint.

```ts
declare function deleteObject(bucket: string, key: string): Promise<{ status: number; headers: Headers }>;

async function retryLater(job: Job, seconds: number): Promise<void> {
  await db.query(
    `UPDATE upload_cleanup_jobs
        SET state = 'pending', run_after = now() + make_interval(secs => $2)
      WHERE id = $1`,
    [job.id, seconds],
  );
}

async function runJob(job: Job): Promise<void> {
  await takeToken();
  try {
    const res = await deleteObject(job.bucket, job.key);
    if (res.status === 429) {
      return retryLater(job, Number(res.headers.get("retry-after") ?? 5));
    }
    // 404: the object is already absent, which is the outcome we wanted.
    if (res.status >= 400 && res.status !== 404) {
      throw new Error(`delete refused with ${res.status}`);
    }
    await db.query(
      `UPDATE upload_cleanup_jobs SET state = 'done', finished_at = now() WHERE id = $1`,
      [job.id],
    );
  } catch (err) {
    if (job.attempts >= 8) {
      await db.query(
        `UPDATE upload_cleanup_jobs SET state = 'dead', last_error = $2 WHERE id = $1`,
        [job.id, String(err)],
      );
      return;
    }
    await retryLater(job, Math.min(2 ** job.attempts, 300));
  }
}

export async function drain(deadlineMs: number, concurrency = 8): Promise<void> {
  const worker = async () => {
    while (Date.now() < deadlineMs) {
      const job = await claim();
      if (!job) return;
      await runJob(job);
    }
  };
  await Promise.all(Array.from({ length: concurrency }, worker));
}
```

Read the retry path once more, because it carries the whole design. A 429 doesn't burn an attempt budget in anger — the row goes back to pending with a delay the store itself suggested. A hard error backs off exponentially, capped at five minutes. After eight tries the row lands in `dead`, where a human can look at it in the morning instead of a pager looking at a human at 03:00. And the deadline argument means the sweep stops when the window closes; unfinished rows are still pending, so tomorrow's run picks them up in order.

The enqueue side stays dull on purpose. One insert per expired object, keyed on the object id, with `ON CONFLICT DO NOTHING` so a scheduler that fires twice produces one job.

## What to watch overnight

Alert on the queue, not on the errors. Error counts spike for reasons that resolve themselves; a queue that stops draining never does.

Four numbers cover most of it: pending depth, the age of the oldest pending job, completions per second, and the count in `dead`. The age of the oldest pending job is the one worth paging on — if the oldest expired upload has been waiting 26 hours, your nightly promise is already broken, no matter how healthy the per-call error rate looks. Completions per second tells you whether the limiter is the constraint or the store is. Emit them as counters and gauges from the worker itself, tag by bucket and region, and keep one log line per terminal transition with the job id, attempt count, and final status. That log is what you'll grep when someone asks why a specific file survived.

Testing gets easier with this shape too. Drain against a fake `deleteObject` that returns a scripted sequence — 429, 500, 404, 204 — and assert the row ends in `done` with the attempt count you expect. That's a plain unit test, no clock mocking, no queue infrastructure.

## Where a queue stops being worth the trouble

Postgres-as-a-queue has a ceiling. It's comfortable into the low thousands of jobs per second on a well-indexed table, and the partial index on `(state, run_after)` matters as much as the query does. Past that, or once fan-out and multi-step orchestration enter the picture, a purpose-built broker earns its operational cost. Celery's documentation is honest about the shape of that trade: a broker plus result backend plus worker fleet is more moving parts than a table, and you take on those parts to get scheduling, routing, and horizontal scale you'd otherwise hand-roll. Stick with the table while the queue is one hop and one owner.

Two objections come up every time.

The first is "why not one big `DELETE` at midnight?" Because the deletes aren't in your database — they're remote calls with their own failure modes and their own rate limits. A single statement can't apply backpressure to an object store, and the transaction it runs in gets long enough to bother your replicas. Do keep the batch delete for the metadata rows, though; that part genuinely is a one-liner, and it should run after the object is confirmed gone.

The second is "why not just use bucket lifecycle rules?" Often you should. If the rule is purely age-based, the store's own expiration does it for free, at a scale no worker pool will match, and I'd take that over a queue every time. It stops being sufficient when deletion depends on state your bucket can't see — a soft-delete flag, a legal hold, a per-tenant retention setting, or the audit record an EU customer will ask for. Lifecycle rules don't emit "who deleted what, and under which policy." A job row does, and it's already sitting there with a timestamp and an attempt count.

Not every scheduled cleanup deserves this much structure. If a sweep touches fifty objects and finishes in a second, a plain cron script is the honest answer, and adding a queue is a cost with no matching benefit. The moment the sweep can be interrupted halfway, or the store can push back, the queue stops being ceremony and starts being the reason the job finishes at all.

## Further reading

- PostgreSQL SELECT documentation, including `FOR UPDATE SKIP LOCKED`: https://www.postgresql.org/docs/current/sql-select.html
- Celery introduction — broker, worker and result backend responsibilities: https://docs.celeryq.dev/en/stable/getting-started/introduction.html
- MDN, `Retry-After` response header: https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Retry-After
- Node.js timers/promises API: https://nodejs.org/api/timers.html
