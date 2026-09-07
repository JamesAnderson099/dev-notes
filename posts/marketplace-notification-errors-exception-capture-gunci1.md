# Marketplace Notification Errors: Exception Capture and Group Search Without Tracing

If you just want to know which order notifications never reached a buyer and why, the smallest setup that answers it is one exception boundary in the send worker, a stable fingerprint, and a store you can search by group. No session replay. No tracing. A marketplace startup whose API fans out email, SMS and push can run on that for a long time before anything richer earns its keep.

The hard part isn't capture. It's deciding which failures deserve to become an error at all — that decision, not the tool, is what keeps a group list readable at 2 a.m.

Here's the field guide I'd hand to a two-person backend team shopping for error monitoring.

| Option | Pick it when | The catch |
| --- | --- | --- |
| Structured logs plus a saved query | Under roughly 20 failures a day, one worker, and somebody already reads logs | No dedup, no resolve state; one broken template shows up 400 identical times |
| A delivery-outcome counter | You want rates, alert thresholds and a weekly trend | It tells you sends are failing, never which line threw |
| Hosted exception tracker with an HTTP ingest endpoint | You want grouping, search and resolve state without operating storage | Events leave your network, and the grouping rules belong to the tool |
| Self-hosted exception tracker | Residency or retention is a contractual requirement, US or EU | You now run a database, an ingest service and a retention job |
| Distributed tracing | A failed notification crosses three or more services and timing is the question | Instrumentation cost is real, and for a single send worker it's not a good fit |

Rows two and three usually get picked together: a counter that says how bad it is, a grouped exception store that says what broke. That pairing is cheap in the only sense that matters here — it keeps both the ingest volume and the human attention small.

## What should a small startup API capture when exception search matters more than tracing?

Classify every send into three outcomes before you decide what to report.

A delivered message is nothing. A rejected message is a business fact about the recipient record: a hard bounce on a typo'd address, an unsubscribe, a push token the device revoked three weeks ago. Nobody can fix those by editing code, so they belong in a counter and in the recipient's own status column — not in an exception group. The third bucket is the interesting one: a template that renders undefined, a provider contract that changed shape, a JSON body your adapter can no longer parse, an auth credential that expired at midnight. Those are defects in your system, and each one should land in exactly one searchable group with a stack trace attached.

Get that split wrong in the noisy direction and the tool stops working within a week. A marketplace with 50k daily notifications and a 2% hard-bounce rate produces 1,000 rejections a day; pipe those into exception capture and every real defect drowns. Your monitoring bill and your team's attention both scale with event volume, which makes classification the cost control, not the plan tier.

Wrong in the quiet direction is worse and slower to notice. If the send worker swallows a provider error and returns success to the API caller, no group is ever created — the queue drains, the dashboard is green, and the seller emails support two days later asking why nobody told them the order shipped. Pair capture with a counter of accepted-versus-completed sends so silence stays visible.

## One boundary, one fingerprint, one group

The shape is easy to say out loud: job leaves the queue, adapter calls the provider, the outcome gets classified, expected rejections increment a counter, and everything unexpected hits a single boundary that fingerprints, redacts, rate limits, then posts.

One boundary. Not one per adapter.

```ts
type Channel = "email" | "sms" | "push";

type Outcome =
  | { kind: "delivered" }
  | { kind: "rejected"; reason: "invalid_address" | "unsubscribed" | "revoked_token" }
  | { kind: "failed"; cause: unknown };

type CapturedEvent = {
  fingerprint: string;
  name: string;
  message: string;
  stack?: string;
  channel: Channel;
  provider: string;
  providerCode: string;
  attempt: number;
  release: string;
  jobId: string;
};

// Everything variable stays out: recipient handle, provider message id, timestamps.
// Two events belong in one group when the same code path fails the same way.
function fingerprintOf(e: {
  channel: Channel; provider: string; providerCode: string; errorName: string;
}): string {
  return [e.channel, e.provider, e.errorName, e.providerCode || "none"].join(":");
}

// droppedEvents is the counter declared in the metrics module further down.
const seen = new Map<string, { count: number; windowStart: number }>();
const WINDOW_MS = 300_000;
const MAX_PER_WINDOW = 20;

function shouldSend(fingerprint: string, now: number): boolean {
  const slot = seen.get(fingerprint);
  if (!slot || now - slot.windowStart > WINDOW_MS) {
    seen.set(fingerprint, { count: 1, windowStart: now });
    return true;
  }
  slot.count += 1;
  return slot.count <= MAX_PER_WINDOW;
}

export async function reportSendFailure(
  ctx: { channel: Channel; provider: string; jobId: string; attempt: number; finalAttempt: boolean },
  cause: unknown,
  now: number,
): Promise<void> {
  if (!ctx.finalAttempt) return;

  const error = cause instanceof Error ? cause : new Error("unclassified send failure");
  const providerCode = String((cause as { code?: string })?.code ?? "");
  const fingerprint = fingerprintOf({ ...ctx, providerCode, errorName: error.name });
  if (!shouldSend(fingerprint, now)) {
    droppedEvents.add(1, { channel: ctx.channel, provider: ctx.provider });
    return;
  }

  const event: CapturedEvent = {
    fingerprint,
    name: error.name,
    message: error.message.slice(0, 300),
    stack: error.stack,
    channel: ctx.channel,
    provider: ctx.provider,
    providerCode,
    attempt: ctx.attempt,
    release: process.env.APP_RELEASE ?? "unversioned",
    jobId: ctx.jobId,
  };

  const res = await fetch(process.env.ERROR_INGEST_URL!, {
    method: "POST",
    headers: {
      "content-type": "application/json",
      authorization: `Bearer ${process.env.ERROR_INGEST_TOKEN}`,
    },
    body: JSON.stringify(event),
  });
  if (res.status === 429) {
    const retryAfter = Number(res.headers.get("retry-after") ?? "5");
    setTimeout(() => void reportSendFailure(ctx, cause, Date.now()), retryAfter * 1000);
  }
}
```

Four decisions in that file carry the whole design. Capture fires on the final attempt only, because a job with five retries otherwise multiplies one defect into five events. The fingerprint is built from channel, provider, error name and provider code — a tuple that stays stable while messages move around. No recipient email, phone number or provider message id ever enters the payload, which keeps the US and EU privacy story short: the events hold system identifiers, and the job id is what joins them to your own logs. And when the per-fingerprint budget is exhausted, the boundary increments a counter of dropped events instead of dropping them silently, so you can still see that a burst happened.

That last one gets skipped constantly. Don't skip it.

The fingerprint rule is where most grouping complaints start. Sentry's grouping documentation describes how stack traces, exception values and explicit fingerprints interact to decide whether two events land in the same group, and the failure mode it warns about is the one a notification service walks straight into: a message string containing a recipient identifier or a provider trace id splits one defect into hundreds of groups. Two events from the same code path, same provider, same error class should be one row in your list. If your tool derives groups from the message by default, override it before you tune anything else.

```ts
import { metrics } from "@opentelemetry/api";

const meter = metrics.getMeter("notification-worker");

// Bounded attributes only. jobId and recipient belong in logs, never here.
const sendOutcomes = meter.createCounter("notification.send.outcomes");
const droppedEvents = meter.createCounter("notification.capture.dropped");

export function recordOutcome(channel: Channel, provider: string, outcome: Outcome): void {
  const kind = outcome.kind;
  const reason = outcome.kind === "rejected" ? outcome.reason : "none";
  sendOutcomes.add(1, { channel, provider, kind, reason });
}
```

Four attributes, all from closed sets. OpenTelemetry's metrics model is built around instruments and aggregations over bounded dimensions, and a counter with an unbounded label is the classic way to turn a cheap metric into an expensive one. Recipient ids go in logs. Rates go in metrics. Defects go in groups.

## Why a resolved group has to be able to come back

Resolution is a state machine, and it's the part teams get wrong after the integration works.

Mark a group resolved when the fix is deployed, never when the ticket is closed — the two events are usually days apart, and a group resolved early quietly hides new occurrences of a live defect. Stamp every event with the release, then treat "same fingerprint, release newer than the one that resolved it" as a regression that reopens the group. That single rule is what makes a resolve button trustworthy enough to keep pressing.

Alerting follows from the same state. Page on a new group and on a regression; don't page on event count, or the first provider incident will wake somebody up 40 times for a failure they can't fix anyway. A weekly review of the top groups by count catches the slow bleed that never crosses a threshold.

Search is not tracing, and the difference matters in incident review. A job id shared between your events and your structured logs lets you jump from a group to the exact log lines for one failed notification. It does not give you a span tree across services, and pretending otherwise wastes an hour during the one incident where you actually needed timing data.

## Where this setup stops paying off

Four honest boundaries, because the answer above is deliberately narrow.

Exception capture doesn't support the questions tracing answers. Once a failed notification is the visible symptom of something three hops upstream — the order service, then a fanout worker, then the send worker — group-and-search leaves you correlating timestamps by hand, and that's the moment to instrument spans rather than argue about tools. Session replay is a different axis entirely; it earns its cost when a browser interaction is needed to reproduce a UI bug, which a backend send worker never has.

Stick with a self-hosted tracker when a customer contract, not a preference, dictates where event data lives and how long it survives. A hosted ingest endpoint in the right region is a deployment detail, and the retention window, the deletion path and the access model are the parts a procurement reviewer will actually ask about. I'm not sure any general recommendation survives that review — your mileage may vary by contract, and the honest answer is to read the data processing terms before the feature matrix.

The last boundary is cultural. A grouped exception store rewards a team that triages weekly and resolves deliberately; left alone for a quarter it becomes an unread inbox with 900 open groups, which is worse than logs because it looks like coverage. If nobody owns the list, the counter and the alert are the only two pieces worth keeping.

## Further reading

- https://opentelemetry.io/docs/concepts/signals/metrics/
- https://docs.sentry.io/concepts/data-management/event-grouping/
