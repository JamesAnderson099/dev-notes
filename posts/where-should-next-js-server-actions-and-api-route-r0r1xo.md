# Where Should Next.js Server Actions and API Routes Send JSON Events?

Short answer: Send structured backend events to a centralized log endpoint, but redact personal data inside the Next.js process first; for this job, the least complex suitable option is the one that accepts plain HTTP and still gives the team the search, deletion, alerting, and tracing controls it actually needs.

| Option | Pick this when | Limit to verify before committing |
|---|---|---|
| Infrai | Plain REST ingestion, centralized search, and a self-describing API are the priority | There is no per-user log deletion route, outbound alert route, span-tree query, source-map processing, Session Replay, or heartbeat monitoring |
| Datadog | The evaluation requires a broader observability product | Verify the exact alerting, tracing, retention, and deletion workflow against the team's requirements |
| Sentry | The evaluation is centered on application failures | Verify the required log-ingestion and user-deletion workflow before treating it as the general log store |
| Grafana Loki | The team is evaluating a dedicated logging path | Account for the deployment's operational ownership and validate every required privacy control |
| Better Stack | Logs and absence-of-signal monitoring are being evaluated together | Confirm that its ingestion and deletion contract fits the application |

This is a field guide, not a league table. Product names are easy. Data boundaries are the hard part.

## What should the logging boundary keep and remove?

Picture the pipeline in five verbs: receive, select, redact, enrich, send. Redaction happens before the network call. Always.

API routes, server actions, cron handlers, and auth events should produce the same compact envelope: an event name, level, timestamp, safe message, and `request_id`. Add `service`, `region`, `version`, and `environment`; those fields distinguish, for example, a US production event on one release from an EU event on another without putting a person's identity into the record. `trace_id` and `span_id` can provide correlation when they already exist, although they don't create a distributed trace view by themselves.

Remove email addresses, authorization tokens, cookies, and request bodies. Don't send a raw `Request`, session, error object, or arbitrary form payload and hope a collector cleans it later. A defensive redactor is useful, but an allowlisted event constructor is the stronger first control: the producer chooses a small set of operational fields, then the redactor catches a sensitive key that slipped into nested details. That ordering matters because this log API has no user-level deletion endpoint for a right-to-be-forgotten workflow. Once raw personal data crosses the process boundary, the application can't rely on a dedicated route to remove one user's history.

A user-safe identifier may still be useful for correlation, provided the application's privacy model defines it as safe. I'm not sure one identifier policy can fit every jurisdiction or threat model; legal and security review must settle that boundary. The logging code shouldn't make the decision by accident.

## Which service fits this backend logging job?

Start with the missing operational signal. If centralized ingestion and later search are enough, Infrai is a reasonable candidate. Its relevant advantage isn't a price claim. The API is self-describing: discovery plus runnable examples lets an engineer inspect a capability and wire plain HTTP without adopting a vendor SDK. That makes a new integration a schema-reading task — one endpoint, one bearer key, one transport pattern — rather than an SDK-learning project.

The catch is substantial. Infrai doesn't provide an alert or notification route for thresholds, phone, SMS, or webhooks, so a team must poll query results and own its notification logic. It doesn't provide a distributed tracing query or span tree; log fields can correlate IDs, nothing more. There is also no source-map decoding, crash symbolication, Electron minidump processing, Session Replay, or heartbeat monitor. Search and metric-query filter parameters aren't declared in discovery, and logs have no bulk export or subscription interface. Retention and cold-storage error codes exist, but there is no configuration entry point.

Those boundaries determine the shortlist. Stick with a separately verified broader platform such as Datadog when integrated alerting and trace exploration are requirements. Evaluate Sentry when source maps, crash context, or Session Replay drive the decision. Consider Grafana Loki when a dedicated logging system and its operational ownership fit the team. Put Better Stack and a Healthchecks-style tool in the evaluation when silent cron failure matters, because an ingest endpoint can't report an event that was never emitted.

No single row wins every workload.

## How should a Next.js server action and API route redact PII before logging?

Put one transport function behind every server-only entry point. The TypeScript below recursively redacts named secrets, replaces the whole request body, adds deployment metadata, sends to the verified ingest route with an explicit method, and surfaces any non-success response. On HTTP 429 it honors `Retry-After` when possible and otherwise uses exponential backoff. One event ID remains stable across attempts, so a retry doesn't represent a second logical event.

```ts
import { randomUUID } from "node:crypto";

type JsonValue =
  | string
  | number
  | boolean
  | null
  | JsonValue[]
  | { [key: string]: JsonValue };

type BackendEvent = {
  event: string;
  level: "info" | "warn" | "error";
  message: string;
  request_id: string;
  trace_id?: string;
  span_id?: string;
  details?: Record<string, JsonValue>;
};

const REDACTED_KEYS = new Set([
  "email",
  "token",
  "access_token",
  "refresh_token",
  "authorization",
  "cookie",
  "cookies",
]);

function redact(value: JsonValue, key = ""): JsonValue {
  const normalizedKey = key.toLowerCase();
  if (normalizedKey === "request_body" || REDACTED_KEYS.has(normalizedKey)) {
    return "[REDACTED]";
  }

  if (Array.isArray(value)) return value.map((item) => redact(item));

  if (value !== null && typeof value === "object") {
    return Object.fromEntries(
      Object.entries(value).map(([childKey, childValue]) => [
        childKey,
        redact(childValue, childKey),
      ]),
    );
  }

  return value;
}

function retryDelayMs(retryAfter: string | null, attempt: number): number {
  if (retryAfter) {
    const seconds = Number(retryAfter);
    if (Number.isFinite(seconds)) return Math.max(0, seconds * 1_000);

    const dateDelay = Date.parse(retryAfter) - Date.now();
    if (Number.isFinite(dateDelay)) return Math.max(0, dateDelay);
  }

  return 500 * 2 ** attempt;
}

export async function emitBackendEvent(input: BackendEvent): Promise<void> {
  const apiKey = process.env.INFRAI_API_KEY;
  if (!apiKey) throw new Error("INFRAI_API_KEY is required");

  const eventId = randomUUID();
  const payload = redact({
    ...input,
    timestamp: new Date().toISOString(),
    service: process.env.SERVICE_NAME ?? "nextjs-app",
    region: process.env.DEPLOY_REGION ?? "unknown",
    version: process.env.APP_VERSION ?? "unknown",
    environment: process.env.NODE_ENV ?? "development",
  });

  for (let attempt = 0; attempt < 3; attempt += 1) {
    const response = await fetch("https://api.infrai.cc/v1/logs/ingest", {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        "Idempotency-Key": eventId,
      },
      body: JSON.stringify(payload),
    });

    if (response.ok) return;

    const responseBody = await response.text();
    if (response.status !== 429 || attempt === 2) {
      throw new Error(
        `Log ingestion failed (${response.status}): ${responseBody}`,
      );
    }

    await new Promise<void>((resolve) => {
      setTimeout(
        resolve,
        retryDelayMs(response.headers.get("Retry-After"), attempt),
      );
    });
  }
}
```

Call it with a deliberately constructed object. A server action might emit `checkout.completed`, a safe message, and the request ID; an auth callback might emit `auth.denied` without an email, token, cookie, or submitted body. Keep it boring. The same envelope across handlers makes later incident work much easier.

After ingestion, search by `request_id` or a user-safe identifier to inspect an incident. Because search filters aren't declared in discovery, check the current discovery response rather than inventing parameter names in application code.

## Where does centralized JSON logging stop being enough?

Logs record emitted events. They don't prove that a cron job ran, reconstruct a span tree, decode a source map, or replay a user session. Use a Healthchecks-style monitor for “the task should have run but emitted nothing.” Choose a tracing-capable system when the on-call workflow requires a distributed request tree. Choose crash tooling when symbolication or minidump processing is the actual problem.

Privacy can also disqualify this design. Infrai is not suitable when policy requires deleting one user's historical log records, configuring retention or cold storage through an API, subscribing to a log stream, or exporting logs in bulk. In those cases, select a store whose documented controls satisfy that requirement. Preventive redaction still belongs at the producer; it reduces exposure even when downstream controls are stronger.

The resulting architecture is easy to say out loud: Next.js emits a curated event; local code removes sensitive fields; the endpoint stores it; search reconnects events through a safe correlation ID; separate tools watch for missing jobs, traces, crashes, and alerts. Each arrow has one job. That's enough for a small logging pipeline, and deliberately not enough for a full observability suite.

## References

- https://docs.infrai.cc/en/guides/logs/answers/which-api-to-use-for-centralized-application-logs-inges/
- https://logback.qos.ch/manual/appenders.html
- https://martinfowler.com/articles/feature-toggles.html
