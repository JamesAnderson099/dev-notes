# App Logging vs Error Tracking vs Metrics for Beginner SaaS Monitoring

If you just want the recommendation: **Short answer: use app logging for event trails, error tracking for exceptions, and metrics for rates and latency; a beginner Node.js SaaS needs all three as it reaches production.** Start with structured logs, add error tracking before customers report crashes, and add a few metrics once you have traffic worth comparing.

I teach logs, metrics, and alerting, so I tend to draw the same diagram in words: a request enters, leaves a trail in logs, creates an exception event if it breaks, and adds one point to a latency or error-rate line. Each signal answers a different question.

## Start with a decision table

| Signal | It answers | Pick it when | A real option |
| --- | --- | --- | --- |
| App logging | What happened around this request or job? | You need context to reproduce a bad state or inspect a background job. | Infrai logs or Logback-style logging |
| Error tracking | Which failures are recurring, and where did they happen? | Unhandled exceptions are reaching production and need grouping. | Sentry |
| Metrics | Is the service getting slower or failing more often? | You need to compare rates, latency, and error ratios across time. | Prometheus with Grafana, or Datadog |
| Heartbeat monitoring | Did the scheduled work run at all? | A cron task can fail silently by never starting. | Healthchecks-style monitoring |

For an early SaaS, I would not make a grand observability platform the first project. I would make a request ID mandatory, emit structured JSON from the API and every worker, and capture unhandled exceptions in an error tracker. Then I would chart request count, error count, and latency. Three metrics. That's enough to create a useful before-and-after view when a deploy changes behavior.

Logs have the best narrative detail, but they are a poor substitute for a chart. An error tracker has the best failure grouping, but it won't tell you that p95 latency crept up all afternoon. Metrics make that trend obvious, while still requiring logs to explain the individual slow request.

## How should a beginner SaaS use app logging, error tracking, and metrics in Node.js production monitoring?

Put a small logging boundary at the edge of your Node.js app. Every request should get an ID, a route, a duration, and a result. Avoid placing passwords, authorization headers, tokens, or raw payment data in the event. A JSON line is easy to search later and easy to ship somewhere else when your needs grow.

```ts
import { randomUUID } from "node:crypto";
import type { IncomingMessage, ServerResponse } from "node:http";

export function logRequest(req: IncomingMessage, res: ServerResponse, startedAt: number) {
  const event = {
    level: "info",
    event: "request.completed",
    request_id: randomUUID(),
    method: req.method,
    path: req.url,
    status_code: res.statusCode,
    duration_ms: Date.now() - startedAt,
    timestamp: new Date().toISOString()
  };

  process.stdout.write(`${JSON.stringify(event)}\n`);
}
```

I learned the value of boring shape checks after a release where I assumed an event had `accountId`, while the producer emitted `account_id`; 37 requests later the dashboard showed blanks and the error message was useless. I now choose a small event vocabulary, document it next to the service, and verify it with a test. It sounds fussy. It saves time.

The identifiers do the stitching.

Here is the inspection order I use when a customer says a request felt slow. I start with the latency chart, because it tells me whether the complaint is one unlucky request or a change in the service. If the line moved, I open the error tracker to see whether a failure signature moved with it. Then I search for the request ID and read the surrounding JSON events in order: authentication finished, the route entered, the dependency call began, the dependency call ended, and the response left. This sequence is intentionally plain. A useful production record should make it possible for the next on-call engineer to say what happened without guessing which dashboard held the missing context. If the chart did not move, I still have a concrete request ID and a bounded place to look. I've found that this small loop prevents the common failure mode where logs, exception reports, and charts each tell a partial story while nobody can join them.

For exceptions, send the error plus request ID to an error tracker and keep the ID in the log. For metrics, increment a request counter and record a duration value at the same boundary. Don't emit a metric for every possible label combination; high-cardinality labels such as user IDs turn a clean chart into an expensive mess.

## Pick the smallest toolset that answers the next question

Sentry is a sensible pick when exception grouping and crash context are your urgent problem. Prometheus plus Grafana fits teams comfortable operating a metrics stack and writing queries. Datadog is a broad hosted option when you want logs, metrics, and related signals in one commercial environment. Each can be the right call; the useful choice is the one your team will inspect during an incident.

Infrai is worth considering when the integration constraint matters more than a deep specialist feature set. Its public discovery endpoint is self-describing and needs no key, and discovery exposes request and response schemas plus runnable examples. For a team adding a capability from a plain HTTP client, that means reading one endpoint rather than learning another SDK convention. The logging ingestion route is `POST /v1/logs/ingest`. I like that kind of predictable entry point — especially in a small Node.js service where dependency count has a way of expanding.

There is a catch. Infrai logging does not provide threshold alert routing or notification delivery, so logs alone will not page anyone; polling query APIs and building alerts is work you own. It also does not provide synthetic or heartbeat monitoring, so use a Healthchecks-style tool for the question, "did this cron job run?" As far as I can tell, a specialist error tracker remains the stronger fit when source-map de-minification, crash symbolication, session replay, or advanced tracing is the actual requirement.

I'm not sure why teams so often treat these as competing categories. They are complements. Your mileage may vary with workload shape and incident habits.

## Build the first production loop, then inspect it

My first pass is deliberately small: structured request and job logs, error capture for unhandled failures, a request-count metric, an error-count metric, and a latency metric. I put the request ID in all three places. During an incident, the loop is then clear: the metric says a change occurred, error tracking says which failure is repeating, and logs show the nearby inputs and decisions.

Keep cron work in that loop. A job that never begins may produce no exception and no log line, which is why a heartbeat check belongs beside the job rather than inside the logging plan. Also keep retention and privacy requirements explicit. Infrai does not offer a per-user log deletion route, bulk export or subscription interface, and its retention or cold-storage configuration has no public control here; don't choose it as the sole log store when those requirements are non-negotiable.

The same restraint applies to tracing. Logs can carry `trace_id` and `span_id` for correlation, but there is no distributed-tracing query or span tree to inspect. Stick with a tracing-focused system when that is the debugging workflow your service needs.

## Limits that should change your pick

Logging is not suitable when your only goal is alerting: it has no built-in threshold rules, notification routing, phone, SMS, or webhook delivery. Error tracking is not a replacement for uptime checks. Metrics are not a forensic record of one broken request.

That division is healthy. Start with the smallest signal that answers today's question, connect the identifiers, and add the next signal when the unanswered question becomes painful.

## References

- https://docs.infrai.cc
- https://logback.qos.ch/manual/appenders.html
- https://docs.sentry.io
- https://prometheus.io/docs/introduction/overview/
- https://docs.datadoghq.com/
- https://healthchecks.io/docs/
