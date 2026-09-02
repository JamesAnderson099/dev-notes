# Cost-Attributed SaaS Delivery: Node.js Readiness, Liveness, and Startup in Docker

In a Node.js media service running in Docker and Kubernetes, readiness and liveness are only useful when a failed delivery can also be assigned to a channel, publication, and cost center.

**Short answer:** give a Node.js process separate startup, readiness, and liveness signals; let Docker and Kubernetes consume those signals; then emit low-cardinality metrics and structured logs that connect delivery outcomes to stable cost-attribution fields. Probes control the workload. Telemetry explains the bill.

Do not make one `/health` response carry every meaning. Startup means initialization has finished. Readiness means the instance may accept delivery work. Liveness means a restart is an appropriate recovery action. A notification provider rejecting one message is a delivery outcome, not proof that the process should restart.

## Test cost attribution before tuning probes

Treat the cost view as an acceptance test, not a dashboard to decorate after launch. Start with a before-and-after picture.

Before: Kubernetes receives a probe response, a worker writes `delivery failed`, and a finance report groups spend by a different set of names. When failures climb, the operator can see pain but cannot tell whether the costly slice is push traffic for a breaking-news publication, email for a weekly digest, or retries caused by a bad recipient address. The platform state, application outcome, and cost record are three disconnected stories.

After: the service uses three narrow probe states and one shared attribution vocabulary. Probe responses decide whether the container starts, receives traffic, or restarts. Delivery events record `channel`, `publication_id`, `cost_center`, `outcome`, and a bounded `failure_class`. Metrics aggregate those same dimensions only where their value sets stay controlled. A dashboard can now compare attempted, delivered, and failed notifications against the units used for internal chargeback.

The diagram in words is short: **probe request -> process state -> status code -> container action**. Beside it runs **delivery attempt -> structured event -> bounded metric labels -> cost view**. The branches meet at workload identity and time, but they do different jobs.

Keep that separation sharp. If a provider rejects a recipient address, count the outcome and log the reason class. Do not fail liveness. If the worker cannot safely accept new jobs during initialization, fail readiness until the required state is available. If the process cannot make progress and restart is the chosen recovery, liveness is the relevant signal. This prevents a business-level failure from turning into a restart loop that creates more retries and muddies attribution.

## Implement one health and delivery vocabulary

Names are architecture here.

Use stable fields for dimensions the team intends to group. `publication_id` and `cost_center` are useful because they connect engineering activity to media operations. `channel` should come from a controlled set such as `email`, `push`, or `sms`. `outcome` can remain similarly small. Raw recipient IDs, message IDs, and provider response text belong in carefully governed logs, not metric labels.

That last boundary matters. An unbounded identifier can create a new metric series for every notification. Even without quoting a vendor-specific limit or price, the operational consequence is clear: a cost-attribution dashboard should aggregate on dimensions that remain enumerable, while detailed investigation uses an event identifier in logs. Don't ask one data shape to serve both accounting and forensic lookup.

The minimal metric set is intentionally plain:

| Signal | Suggested dimensions | Question answered |
| --- | --- | --- |
| `notification_attempts_total` | channel, publication, cost center | Where is delivery work occurring? |
| `notification_outcomes_total` | channel, publication, cost center, outcome | Which attributed slice is failing? |
| `notification_retry_total` | channel, publication, cost center, failure class | Which failure classes amplify work? |
| `worker_ready` | workload identity | Can this instance accept work now? |

Logs add the context that counters deliberately omit: event ID, timestamps, a redacted destination reference, and the normalized failure class. A `401` from a downstream request and a `429` describe different actions, for example, but neither should be copied blindly into a metric label with arbitrary response text. Normalize first.

Cost itself may arrive later from a billing export or internal rate table. Join that data to the same controlled channel and cost-center vocabulary rather than pretending that an application counter is an invoice. I'm not sure which source is authoritative in every organization; the decision belongs to finance and engineering together, and a reconciliation using one closed reporting period will expose mismatched definitions.

## How can a Node.js app connect probe metrics and logs?

This Node.js example uses the standard HTTP server. It keeps probe state separate from notification outcomes, writes structured events, and exposes a JSON snapshot that a reporting adapter can translate into the team's metric format. The values are illustrative state, not measured production results.

```ts
import { createServer, IncomingMessage, ServerResponse } from "node:http";

type Probe = "startup" | "readiness" | "liveness";
type Channel = "email" | "push" | "sms";
type Outcome = "delivered" | "rejected" | "retryable";

type Dimensions = {
  channel: Channel;
  publicationId: string;
  costCenter: string;
  outcome: Outcome;
};

const health = { started: false, ready: false, live: true };
const counters = new Map<string, number>();

function metricKey(dimensions: Dimensions): string {
  return [
    dimensions.channel,
    dimensions.publicationId,
    dimensions.costCenter,
    dimensions.outcome,
  ].join(":");
}

function recordDelivery(
  eventId: string,
  dimensions: Dimensions,
  failureClass?: "authentication" | "rate_limited" | "invalid_recipient",
): void {
  const key = metricKey(dimensions);
  counters.set(key, (counters.get(key) ?? 0) + 1);

  process.stdout.write(`${JSON.stringify({
    timestamp: new Date().toISOString(),
    event: "notification_delivery_completed",
    event_id: eventId,
    channel: dimensions.channel,
    publication_id: dimensions.publicationId,
    cost_center: dimensions.costCenter,
    outcome: dimensions.outcome,
    failure_class: failureClass,
  })}\n`);
}

function probeValue(probe: Probe): boolean {
  if (probe === "startup") return health.started;
  if (probe === "readiness") return health.ready;
  return health.live;
}

function sendJson(response: ServerResponse, status: number, body: unknown): void {
  response.writeHead(status, { "content-type": "application/json" });
  response.end(JSON.stringify(body));
}

function route(request: IncomingMessage, response: ServerResponse): void {
  const match = request.url?.match(/^\/health\/(startup|readiness|liveness)$/);
  if (request.method === "GET" && match) {
    const probe = match[1] as Probe;
    const healthy = probeValue(probe);
    sendJson(response, healthy ? 200 : 503, { probe, healthy });
    return;
  }

  if (request.method === "GET" && request.url === "/internal/metrics-snapshot") {
    sendJson(response, 200, {
      worker_ready: health.ready ? 1 : 0,
      notification_outcomes_total: Object.fromEntries(counters),
    });
    return;
  }

  sendJson(response, 404, { error: "not_found" });
}

const server = createServer(route);

server.listen(3000, () => {
  health.started = true;
  health.ready = true;

  recordDelivery("evt_example", {
    channel: "push",
    publicationId: "daily-news",
    costCenter: "editorial-alerts",
    outcome: "delivered",
  });
});

process.on("SIGTERM", () => {
  health.ready = false;
  server.close(() => process.exit(0));
});
```

The example returns `503` while a probe state is false and `200` while it is true. Point each Kubernetes probe at its matching path. Keep timing and thresholds in deployment configuration because initialization time and recovery policy are properties of the workload, not universal constants. Docker can use the same narrow endpoint for its container health check, while Kubernetes retains the distinct startup, readiness, and liveness decisions.

Notice what the code refuses to do. A rejected push does not change `health.live`. The delivery record increments an attributed outcome and emits one structured event. During termination, readiness becomes false before the server closes, making traffic eligibility explicit. Small distinction. Big operational payoff.

The snapshot endpoint is not a public API. Restrict it to the monitoring path, and have the collection layer convert it to the established metrics system. In a larger service, counters would normally live behind a metrics library rather than a process-local map; the map keeps this example focused on semantics. Process-local counters also reset with the process, so the collector must retain the time series if historical totals matter.

## Test failure behavior before trusting the dashboard

Happy-path screenshots prove very little. Exercise the transitions that operators will need to explain.

First, delay initialization and verify startup remains false without allowing liveness to terminate a process that is still legitimately starting. Next, make the worker temporarily unable to accept jobs and verify readiness removes it from work without representing the condition as process death. Then simulate a delivery rejection and a retryable outcome; confirm that neither changes liveness, both preserve their cost-attribution fields, and each increments exactly one outcome counter.

Also test shutdown. Send `SIGTERM`, observe readiness become false, and verify the server stops accepting work before exit. The precise grace period depends on the deployment and queue semantics, so it should be tested with the actual workload rather than copied from an example.

For cost attribution, use a small fixture spanning at least two publications, channels, and cost centers. The exact quantities do not matter. What matters is conservation: the sum of grouped outcomes must equal the accepted attempts under the team's documented retry definition. Decide whether each retry is a new billable attempt or part of one logical notification before naming the counter. Otherwise two correct systems can show different totals and both appear broken.

Feature flags need similar discipline. A channel rollout can be separated from deployment, but the flag state should be captured with the delivery decision when it changes routing or spend. Do not place arbitrary flag keys in metric labels. Record a controlled rollout cohort or routing mode, and keep the detailed flag evaluation in logs. The distinction between deploying code and releasing behavior is described in Martin Fowler's feature-toggle reference.

Privacy is part of this test plan — especially for rejection logs. A recipient address is not required for cost allocation, so prefer a redacted reference and define deletion behavior for any user-linked event data. GDPR Article 17 makes erasure a concrete workflow concern, not a dashboard footnote.

## Where does this simple monitoring design stop?

The design is a good fit for a small service that needs container control signals, delivery-failure triage, and internal cost allocation from a shared vocabulary. It stays portable because the application emits structured output and exposes narrow health states instead of depending on a commercial ingestion contract.

The catch is scope.

It is not suitable by itself when the acceptance test requires external uptime checks from multiple locations, native alert delivery, cross-service trace exploration, long-term metric storage, or audited billing reconciliation. Use dedicated components for those jobs. Likewise, stick with an existing organization-wide telemetry pipeline when it already provides collection, retention, access control, and alert routing; adding a second monitoring destination would split the incident record.

There is also a deliberate privacy trade-off. More log context can shorten diagnosis, yet recipient data expands the deletion and access-control burden. Keep cost dimensions separate from personal identifiers, document retention, and test erasure. That is less exciting than another chart. It is more durable.

A useful rollout gate has four checks: each probe triggers only its intended container action; every delivery outcome retains channel, publication, and cost center; metric labels remain controlled; and grouped totals reconcile under one written retry rule. Pass those checks before adding panels. The result is simple health monitoring with an honest boundary: Kubernetes operates the workload, while logs and metrics explain notification failure and cost.

## References

- https://martinfowler.com/articles/feature-toggles.html
- https://gdpr-info.eu/art-17-gdpr/
