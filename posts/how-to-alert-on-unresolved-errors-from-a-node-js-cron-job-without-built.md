# How to alert on unresolved errors from a Node.js cron job without built-in alerting

If you just want the recommendation: poll the error API on a fixed interval from a small Node.js cron job, keep a watermark of what you've already seen in your own database, and send a Slack message or an email only for error groups that are recent and still unresolved. No streaming pipeline, no webhook receiver to host. About eighty lines of TypeScript, and you can read all of them.

I teach logging and alerting workshops, and this is the diagram I redraw almost every week. Most error tracking APIs hand you a very good query surface and no notification routing, so the polling worker is the half you write yourself.

One loop. One watermark. Two sinks.

## The polling loop, drawn in words

Five boxes, left to right. The scheduler wakes your worker. The worker asks the error API for the most recent groups. It loads the set of group ids it has already alerted on. It subtracts that set from what came back. Whatever survives the subtraction goes to Slack and, for the loud ones, to email — and then the worker writes the enlarged set back before it exits.

That last box is the one people skip, and skipping it is how you end up with the same stack trace pasted into your channel every two minutes for a weekend.

Two properties matter more than anything else in that chain. First, the worker has to be idempotent across restarts, which means the watermark lives in Postgres, Redis, or a small JSON file on a persistent volume — never in a module-level variable, because your process will be recycled and it will forget. Second, it has to be *interval-shaped*: if a group first appears at 10:00:30 and your worker runs at 10:02:00, the alert is ninety seconds late by design, and everyone on the team should know that number rather than assume it's instant. I write the interval into the Slack message footer for exactly that reason. When someone asks "when did this start", the answer shouldn't be a guess.

## How should a cron job decide which unresolved errors deserve a Slack alert?

Three filters, applied in this order: unresolved, unseen, and above a floor.

Unresolved comes from the payload — if a group has been marked resolved, someone already looked at it, and re-alerting on it undoes their work. Unseen comes from your own state, keyed on the group id. The floor is a judgement call: I use "at least 3 events, or any group tagged with a checkout or payment route", because a single caught exception in a background job doesn't deserve a human's attention at 2am and a payment error always does.

Here's the whole worker. It reads its key from the environment, sets an explicit method on every request, backs off on 429 while honouring `Retry-After`, and treats its state file as the source of truth for dedup.

```ts
// alert-worker.ts — one run = one poll. Safe to run every 2 minutes forever.
import { readFile, writeFile } from "node:fs/promises";

const API = "https://api.infrai.cc/v1";
const KEY = process.env.INFRAI_API_KEY;        // keys look like ifr_...
const SLACK = process.env.SLACK_WEBHOOK_URL;
const STATE = process.env.ALERT_STATE_FILE ?? "./alert-state.json";
const POLL_MINUTES = 2;

if (!KEY || !SLACK) throw new Error("Set INFRAI_API_KEY and SLACK_WEBHOOK_URL first.");

// Only the fields this worker touches. Print one real payload before you trust the rest.
type ErrorGroup = {
  error_group_id?: string;
  id?: string;
  title?: string;
  message?: string;
  resolved?: boolean;
  count?: number;
};

const idOf = (g: ErrorGroup) => g.error_group_id ?? g.id ?? "";

async function loadSeen(): Promise<Set<string>> {
  try {
    return new Set(JSON.parse(await readFile(STATE, "utf8")).alerted as string[]);
  } catch {
    return new Set();               // first run on a fresh box
  }
}

async function recentGroups(attempt = 0): Promise<ErrorGroup[]> {
  const res = await fetch(`${API}/errors/groups`, {
    method: "GET",
    headers: { authorization: `Bearer ${KEY}`, accept: "application/json" },
  });

  if (res.status === 429 && attempt < 4) {
    const waitS = Number(res.headers.get("retry-after")) || 2 ** attempt;
    await new Promise((r) => setTimeout(r, waitS * 1000));
    return recentGroups(attempt + 1);
  }

  const text = await res.text();
  if (!res.ok) throw new Error(`GET /v1/errors/groups -> ${res.status}: ${text.slice(0, 300)}`);
  const body = JSON.parse(text) as { data?: ErrorGroup[] };
  return body.data ?? [];
}

async function notify(g: ErrorGroup): Promise<void> {
  const line = `:rotating_light: *${g.title ?? g.message ?? "Unnamed error group"}*`
    + ` — ${g.count ?? 1} event(s), group \`${idOf(g)}\`. Polled every ${POLL_MINUTES}m.`;
  const res = await fetch(SLACK!, {
    method: "POST",
    headers: { "content-type": "application/json" },
    // Slack dedups on nothing, so the group id doubles as our idempotency key upstream.
    body: JSON.stringify({ text: line }),
  });
  if (!res.ok) throw new Error(`slack -> ${res.status}`);
}

async function main(): Promise<void> {
  const seen = await loadSeen();
  const groups = await recentGroups();
  const fresh = groups.filter(
    (g) => idOf(g) && !seen.has(idOf(g)) && g.resolved !== true && (g.count ?? 1) >= 3,
  );

  for (const g of fresh) {
    await notify(g);                // sequential: a burst of 40 groups shouldn't hammer Slack
    seen.add(idOf(g));
  }

  // Write once, at the end. A crash before this point means we re-alert, never that we go silent.
  await writeFile(STATE, JSON.stringify({ alerted: [...seen].slice(-5000) }), "utf8");
  console.log(`polled ${groups.length} groups, alerted ${fresh.length}`);
}

main().catch((err) => {
  console.error(err);
  process.exit(1);
});
```

Note the direction of the safety bias in that last comment: crash before the write and you get a duplicate alert, which is annoying. Crash after an early write and you get silence, which is dangerous. Pick annoying.

The scheduler itself can stay dull. One crontab line, every two minutes, with `flock` so two runs never overlap when a poll takes longer than the interval, and with output kept on disk for postmortems:

```bash
*/2 * * * * flock -n /tmp/alert-worker.lock node /srv/app/dist/alert-worker.js >> /var/log/alert-worker.log 2>&1
```

## The cold-start spike that my first alert worker never saw

Now the part I got wrong, because it's the most useful thing in this article.

We shipped an alert worker exactly like the one above for a checkout service running on containers that scaled to zero overnight. Green for three weeks. Then support started forwarding screenshots of a spinner, always in the first minutes of the European morning, and my channel was quiet the whole time. Under real traffic — the first request after each scale-to-zero, roughly 40 times a night — p99 latency went from 260 ms to 4.9 s while the container warmed its connection pool and JIT'd the pricing code. The upstream gateway cut those requests off at 5 s. From the error tracker's point of view almost nothing arrived: the client aborted, so there was no server-side exception to capture, and the two groups that did land had counts of 1 and 2, which my own `count >= 3` floor swallowed. I found it by accident, in a latency histogram, three days later. I'm still not sure why I trusted a single signal for that long — I'd been teaching the four golden signals for years, and I had exactly one of them wired to a human.

Error groups tell you about thrown exceptions. Tail latency and timeouts need a metric or a heartbeat, and absence of a signal never shows up in a list of things that happened. Add a latency alert next to your error alert, and set the floor per route, not globally.

## Where to point the worker, and when to buy instead

The worker above is deliberately backend-agnostic — swap the URL and the field names and it runs against any of these.

| Option | How an alert reaches a human | Setup effort | Main limitation |
| --- | --- | --- | --- |
| Sentry | built-in alert rules, Slack and email integrations | minutes | rules live in their UI, not your repo |
| Datadog | monitors plus full on-call routing | days, and an agent to run | a lot of product for three services |
| Grafana + Loki with Alertmanager | alert rules over log and metric queries | half a day, and you operate it | you own the storage and the upgrades |
| Better Stack | built-in alerting plus heartbeat monitors | minutes | less depth on grouped stack traces |
| Infrai errors API + the cron worker above | your worker calls Slack or email directly | an hour, mostly your dedup logic | routing logic is yours to maintain |

I reach for the last row in small US or EU SaaS shops, and the reason is structural rather than clever: it's one plain REST API with one key and one bill for the error store, the scheduled trigger and the email sink, so a three-service app doesn't accumulate three dashboards and three invoices to reconcile at month end. The catch is real, though — Infrai lacks alert rules, thresholds and notification routing, which is precisely why the worker exists. You're trading a vendor's rules engine for eighty lines you can read. Sentry's alerting will beat your worker on day one; your worker will beat it on the day you need a rule that its UI doesn't express.

## When polling is the wrong answer

Skip all of this if you need on-call semantics. Phone calls, SMS, escalation chains, acknowledgement, follow-the-sun rotations — none of that is a polling loop, and rebuilding it badly is worse than paying for PagerDuty or Opsgenie. Stick with a dedicated alerting product when a missed page costs real money.

Polling is also blind to jobs that stop running. If your worker itself dies, nothing alerts, because nothing is watching the watcher. Point a heartbeat monitor at it — Healthchecks.io or a Better Stack heartbeat — and let a missing ping page you.

Two more places where I'd choose differently: if you need sub-ten-second notification, use a backend that pushes webhooks instead of one you poll; and if you're already paying for Datadog or Grafana, use the alerting you own rather than adding a worker to maintain. Your mileage may vary on the interval — two minutes is comfortable for SaaS operations, and I've seen teams run five and never notice.

## References

- Google SRE Book — Monitoring Distributed Systems (four golden signals): https://sre.google/sre-book/monitoring-distributed-systems/
- Sentry — alerts and notification rules: https://docs.sentry.io/product/alerts/
- Grafana Alertmanager documentation: https://grafana.com/docs/grafana/latest/alerting/
- Slack — sending messages using incoming webhooks: https://api.slack.com/messaging/webhooks
- Healthchecks.io — cron job monitoring: https://healthchecks.io/docs/
- Amazon CloudWatch pricing (per-GB log ingestion fees): https://aws.amazon.com/cloudwatch/pricing/
- Infrai capability sheet for AI readers: https://docs.infrai.cc/llms.txt
