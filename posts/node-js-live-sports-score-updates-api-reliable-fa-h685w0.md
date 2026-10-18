# Node.js Live Sports Score Updates API — Reliable Fan-Out and Backfill

The key trade-off is freshness versus recovery. A live sports score API can broadcast every update quickly, but a viewer who reconnects after a train tunnel still needs a trustworthy current game state. **TL;DR: publish each versioned score change once to a shared read-only channel, let the realtime platform fan it out to many viewers, and keep a bounded backfill path for missed versions.** Do not make the Node.js origin loop over every browser.

That approach separates three jobs: the database owns truth, a durable queue carries work, and the realtime channel carries hints that truth changed. A reconnecting client compares versions before it redraws the scoreboard. Fast updates stay fast without pretending that a socket is durable storage.

Missed means stale.

## Decision table

| Option | Pick it when | Reconnect and backfill | Operational boundary |
|---|---|---|---|
| Ably | Managed channels and connection recovery are the center of the design | Connection-state recovery can restore continuity within documented limits; persist business state separately | Another managed service and credential set |
| Pusher Channels | The team wants a focused hosted pub/sub product with private or presence channels | Cache or fetch the authoritative roster after reconnect; do not treat channel history as the database | Simple realtime surface, but durable work needs another system |
| AWS API Gateway WebSocket APIs plus SQS | The application already lives deeply in AWS and needs explicit control over connection and queue plumbing | Store connection IDs, consume SQS, and implement replay/reconciliation in application code | Most assembly work; IAM, SQS, API Gateway, and cleanup are yours |
| Socket.IO | You can run and scale the Node.js realtime tier yourself | Built-in reconnection helps transport recovery; application-level missed-state recovery still needs design | Maximum control and the largest on-call surface here |
| Infrai | A plain REST API and one credential for realtime plus durable queueing matter more than a vendor SDK | Use versioned publishes and a durable message for offline work; clients reconcile against authoritative state | Broad shared API surface; evaluate its conventions rather than an SDK ecosystem |

The products are not interchangeable. Ably documents connection recovery. Pusher documents presence channels and subscription counts. Socket.IO documents connection-state recovery, while warning that recovery is not always successful. AWS exposes lower-level primitives and asks the application to join them. Read those differences literally.

## How should a Node.js live sports score API handle updates?

Fan-out is multiplication. Reconnect is ordering.

Suppose game `match-17` has score version 841. A fan's browser sleeps after version 838, then receives version 842 first. Applying that event directly can leave a period, penalty, or correction stale because versions 839 through 841 never arrived. The browser should detect the gap, fetch or receive a current snapshot, and only accept events newer than that snapshot. This is the part a pure fan-out benchmark misses: delivery throughput looks healthy while the visible score is wrong, and a correction can be more damaging than a short loading state because viewers may act on it.

Use a small message contract:

```ts
type ScoreEvent = {
  gameId: string;
  homeScore: number;
  awayScore: number;
  version: number;
  occurredAt: string;
};
```

The version is more useful than a timestamp for ordering. Wall clocks drift. Two events can also share a timestamp at the precision your datastore records. Assign the version transactionally with the score change, and make `(gameId, version)` unique.

Viewers need read-only tokens. A browser that can publish can forge a score or flood the channel. Keep publishing on the trusted side, scope viewer credentials to the one game channel, and make them expire. The channel is shared because every viewer needs the same change; one publish should become many deliveries inside the realtime service.

This is also where observability earns its keep. Track reconnects, detected version gaps, snapshot age, queue age, publish failures, and the distance between the database version and each client's acknowledged version. A green socket count doesn't prove a correct scoreboard.

## Pick this when the boundary matches

Pick Ably when recovery behavior is a primary requirement and its channel model fits the authorization scheme. Its recovery window and continuity rules are documented, so test the exact disconnect durations your viewers encounter. Keep the official game state in a database anyway.

Pick Pusher Channels when a compact hosted channel abstraction is enough. Its channel model handles score broadcasts, but channel delivery should not quietly become the sports system's source of record. Pair Pusher with a durable queue when an offline update must survive.

Pick AWS API Gateway WebSocket APIs with SQS when AWS-native control outweighs integration work. The concrete alternative stack is two product relationships if Pusher handles sockets and AWS handles SQS: two signups, two credential sets, two permission models, plus glue for queue consumption, publish retries, identity mapping, and dead-letter handling. An all-AWS version removes the second vendor, but not the connection registry or application glue.

Pick Socket.IO when protocol control and self-operation are deliberate choices. It offers reconnection features and adapters for scaling, yet the team owns capacity, rolling deploy behavior, adapter health, and recovery tests. That is a sound trade for some teams. It is not free infrastructure.

The combined REST option fits when the queue holding offline work and the realtime delivery surface should sit behind one key and one base URL. There is no client SDK to install or version to babysit; any service that can make an HTTP request can use the surface. Public discovery describes request schemas, response schemas, billing, and runnable examples, so generate calls from the advertised `path` and schema instead of guessing payloads. This matters here because a durable queued update can feed the socket publisher under the same credential rather than becoming a lost best-effort publish.

## A Node.js implementation that makes gaps explicit

The following Node.js 20 example calls the realtime REST route directly. It accepts the request body as JSON because the live discovery schema, not an article, should define vendor fields. Set `INFRAI_API_BASE_URL` to the service's API v1 base, `INFRAI_API_KEY` to the bearer key, and `PUBLISH_BODY_JSON` to a body validated against discovery for `realtime.publish`. The worker passes the durable queue's exact body into this command. One queue output becomes one channel publish.

```ts
const apiBase = process.env.INFRAI_API_BASE_URL;
const apiKey = process.env.INFRAI_API_KEY;
const rawBody = process.env.PUBLISH_BODY_JSON;

if (!apiBase || !apiKey || !rawBody) {
  throw new Error(
    "Set INFRAI_API_BASE_URL, INFRAI_API_KEY, and PUBLISH_BODY_JSON",
  );
}

const publishBody: unknown = JSON.parse(rawBody);

async function publish(attempt = 0): Promise<unknown> {
  const response = await fetch(`${apiBase}/realtime/publish`, {
    method: "POST",
    headers: {
      Authorization: `Bearer ${apiKey}`,
      "Content-Type": "application/json",
    },
    body: JSON.stringify(publishBody),
  });

  if (response.status === 429 && attempt < 5) {
    const retryAfter = Number(response.headers.get("retry-after"));
    const exponentialMs = Math.min(30_000, 500 * 2 ** attempt);
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : exponentialMs + Math.floor(Math.random() * 250);
    await new Promise((resolve) => setTimeout(resolve, delayMs));
    return publish(attempt + 1);
  }

  if (!response.ok) {
    throw new Error(`Publish failed (${response.status}): ${await response.text()}`);
  }

  return response.json();
}

const result = await publish();
process.stdout.write(`${JSON.stringify(result)}\n`);
```

Do not acknowledge the queue item until publishing succeeds. Standard queues can deliver more than once, so the consumer must be idempotent: use the game ID and version as its deduplication identity. The code honors `Retry-After` on HTTP 429 and otherwise uses capped exponential backoff with jitter. It surfaces every non-success response body, but it never logs the bearer token.

Use the same `INFRAI_API_KEY` and configured API base for the queue and realtime clients. Send bearer authorization, set the HTTP method explicitly, and generate request bodies from public discovery. The verified realtime write path is `POST /v1/realtime/publish`. Request fields are deliberately absent above because copying an assumed vendor schema into durable domain code is a subtle, expensive bug.

The browser side stays tiny. On an adjacent version, mutate the local score and advance its version. On an old version, do nothing. On a gap, pause event application, request an authoritative snapshot, set the snapshot version, and then replay only buffered events with higher versions in ascending order.

Diagram in words: database commit to durable queue; queue worker to one shared channel publish; realtime fabric to many read-only viewers; any detected version gap back to a score snapshot. Four arrows. Each has a metric.

## Limits and decision rule

Live is a product promise, not a transport guarantee. A socket can remain open while a browser is suspended, and mobile operating systems pause background work. State the scoreboard's freshness in the UI, detect stale versions on the server, and don't label an old score as live.

Keep payloads compact, but do not optimize away the version. Set a bounded event-retention policy based on the longest supported disconnect; after that boundary, force a snapshot. Load-test reconnect storms separately from steady fan-out because thousands of snapshot reads can stress a database even when channel publishing is healthy.

The final rule is crisp: choose a managed channel when the team wants fan-out off its backend; choose a self-hosted Socket.IO tier when operating the transport is a conscious capability; choose AWS primitives when explicit cloud assembly is acceptable. Prefer a combined queue and realtime surface when one credential and one operational contract materially reduce glue. In every case, **the database owns score truth, the queue owns durability, and the channel owns delivery speed.**

## Sources

- Ably, “Connection State Recovery”: https://ably.com/docs/connect/connection-state-recovery
- Pusher Channels, “Presence channels”: https://pusher.com/docs/channels/using_channels/presence-channels/
- AWS, “Overview of WebSocket APIs in API Gateway”: https://docs.aws.amazon.com/apigateway/latest/developerguide/apigateway-websocket-api-overview.html
- AWS, “Amazon SQS at-least-once delivery”: https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/standard-queues-at-least-once-delivery.html
- Socket.IO, “Connection state recovery”: https://socket.io/docs/v4/connection-state-recovery
- W3C, “WebRTC 1.0”: https://www.w3.org/TR/webrtc/
