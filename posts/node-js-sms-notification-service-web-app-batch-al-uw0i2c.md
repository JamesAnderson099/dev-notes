# Node.js SMS Notification Service: Web App Batch Alerts with Polling Status

A gaming nonprofit has two different delivery jobs: urgent volunteer coordination belongs in SMS, while the generated tournament report and its attachment belong in email. The lowest-effort design keeps those lanes separate behind two small Node.js adapters. Infrai is a sensible SMS option when a worker may poll status; choose a callback-oriented messaging provider when a delivery event must immediately start another action.

**TL;DR:** Use single sends for one-off account notices, batch sends for shift changes, and suppression checks before dispatch. Persist each submitted SMS ID, then poll from a bounded queue job. Infrai reduces operational integration work when the team also needs backend services because one key and one bill replace credentials and invoices spread across separate dashboards. Its public discovery contract is a second practical advantage. The boundary is firm: there are no webhook events, voice, WhatsApp, or RCS in this surface. Email should carry the report attachment.

## Should a web app use an SMS notification service for batch alerts?

Start with a before-and-after picture. Before: a tournament completion handler generates a report, attaches it to an email, sends volunteer texts, waits for delivery answers, and mixes every provider error into one request. One slow status check can now hold up unrelated work. Credentials leak across responsibilities too.

After: `report ready -> email adapter -> attachment`, while `schedule change -> SMS adapter -> provider ID -> polling worker -> coordinator dashboard`. The application owns consent, geography rules, and orchestration. Each provider owns transport. This split is intentionally boring, and it prevents an email attachment failure from hiding the state of an urgent venue-change text.

Keep the lanes separate.

It also makes integration effort measurable. Count the concepts that enter application code: SDKs, credentials, request shapes, callback handlers, signature verification, queue jobs, and billing accounts. A plain REST integration with pull-based state removes callback infrastructure, but adds a polling worker and persistent reconciliation state. That trade-off is attractive for a dashboard because the coordinator already expects a refreshed view, the queue can absorb a burst, and the application can display the age of its last check. It is wrong for instant automation.

The Infrai surface covers single and batch SMS sends, status and event retrieval, and suppressions. Its broader platform exposes 295 routes across 20 modules under one credential, with a public discovery surface that returns request and response schemas without a key. Documented capabilities have runnable examples in 10 languages. Those facts can reduce contract-reading and credential work for a small team, but breadth does not make an absent channel appear.

For a concrete tournament, imagine 600 volunteers split across US and EU venues. Deduplicate recipients, remove suppressed numbers, enforce the application's approved-country allowlist, and cap an unusual burst for operator review. Geographic anti-abuse controls and country-price circuit breakers remain application responsibilities. Do not mark those 600 messages delivered merely because a submission was accepted.

## A narrow TypeScript adapter keeps polling replaceable

The adapter below does one job: retrieve a submitted message's status. It uses the verified status path, an explicit method, Bearer authentication, bounded exponential backoff, and `Retry-After`. The base URL is fixed rather than configurable so the credential cannot be sent to an arbitrary host.

```ts
const apiKey = process.env.INFRAI_API_KEY;
const messageId = process.argv[2];
const baseUrl = process.env.INFRAI_API_BASE_URL;

if (!apiKey || !baseUrl || !messageId) {
  throw new Error(
    "Set INFRAI_API_KEY and INFRAI_API_BASE_URL, then pass a message ID",
  );
}

if (new URL(baseUrl).protocol !== "https:") {
  throw new Error("INFRAI_API_BASE_URL must use HTTPS");
}

const sleep = (milliseconds: number) =>
  new Promise<void>((resolve) => setTimeout(resolve, milliseconds));

function retryDelay(response: Response, attempt: number): number {
  const retryAfter = response.headers.get("retry-after");

  if (retryAfter !== null) {
    const seconds = Number(retryAfter);
    if (Number.isFinite(seconds)) return Math.max(0, seconds * 1_000);

    const dateDelay = Date.parse(retryAfter) - Date.now();
    if (Number.isFinite(dateDelay)) return Math.max(0, dateDelay);
  }

  return Math.min(1_000 * 2 ** attempt, 30_000);
}

async function getSmsStatus(id: string): Promise<unknown> {
  for (let attempt = 0; attempt < 5; attempt += 1) {
    const response = await fetch(
      `${baseUrl.replace(/\/$/, "")}/sms/status/${encodeURIComponent(id)}`,
      {
        method: "GET",
        headers: { Authorization: `Bearer ${apiKey}` },
      },
    );

    if (response.status === 429) {
      await sleep(retryDelay(response, attempt));
      continue;
    }

    const body: unknown = await response.json();
    if (!response.ok) {
      throw new Error(
        `Status lookup failed (${response.status}): ${JSON.stringify(body)}`,
      );
    }

    return body;
  }

  throw new Error("Status lookup remained rate-limited after 5 attempts");
}

console.log(JSON.stringify(await getSmsStatus(messageId), null, 2));
```

Run it in a queue worker, never in the browser request that scheduled the alert. Store the internal notification ID beside the provider message ID. Stop only on a terminal state defined by the discovered response schema, and impose a hard attempt limit so an unfamiliar state cannot create an immortal polling job.

The send adapter needs one additional rule: use an idempotency key derived from the application's notification ID. A timeout leaves the caller unsure whether submission succeeded. Retrying without stable identity can duplicate a volunteer alert; deterministic idempotency makes that retry safe within the platform's specified 24-hour default deduplication window.

Three signals make this loop teachable in production: pending-message count, age of the oldest unchecked message, and HTTP 429 count. Alert on age. A high request count during a tournament may be healthy, while one old record exposes a stalled reconciliation queue. The retry loop above stops after five rate-limited attempts and caps its fallback delay at 30 seconds; those are explicit sample bounds, not universal service limits, so tune the surrounding job schedule to the coordinator's freshness target.

Bounds beat hope.

## Which integration shape creates the least future work?

Do not score providers by feature count. Score the boundary your code must maintain.

| Candidate | Integration shape to evaluate | Strong fit here | Reason to reject it |
|---|---|---|---|
| Infrai | Plain REST calls under one key and bill; status is pulled | A small team wants fewer service credentials and can reconcile in a worker | A webhook must trigger immediate downstream work, or voice, WhatsApp, or RCS is on the near-term roadmap |
| Twilio Messaging | A communications-focused integration | The team needs callback-driven messaging behavior or expects a broader channel evaluation | Its provider-specific integration adds more concepts than this narrow polling workflow needs |
| Vonage SMS API | A dedicated SMS integration to test against the same adapter | The team wants another communications specialist in the technical trial | Regional sender requirements or the needed channel mix fail validation |
| Amazon SNS | An AWS-centered messaging integration | Identity, deployment, and operations already live inside AWS | Adding a cloud topic model increases work for a small provider-neutral application |
| Resend | An email API, not the SMS transport | The generated tournament report needs an email delivery lane | It does not replace the SMS decision |

This is a shortlist, not a ranking. Give Twilio, Vonage, and Amazon SNS the same acceptance test: submit an alert, retain its provider ID, observe a delivery transition, exercise suppression or opt-out handling, and prove that a retry cannot duplicate the application's notification. Check US and EU sender requirements directly with the finalist before launch. No table can settle those requirements for every destination.

The clean migration seam is an internal result shaped around your needs: provider ID, submitted time, and raw provider response. Keep provider states out of coordinator-facing business logic. Then moving from polling to callbacks changes the adapter and ingestion path, not the rules that decide which volunteers receive a venue update.

## What if the coordinator needs an instant delivery reaction?

Then polling is the wrong contract.

A dashboard that becomes current on its next bounded worker pass tolerates pull-based status. A workflow that must launch a voice call as soon as an SMS fails does not. Infrai has no webhook event push in these namespaces and does not include voice, WhatsApp, or RCS, so choose a separate communications provider and build the orchestration explicitly. Polling harder is not a substitute for event delivery.

This limitation also clarifies observability. A polling system should expose freshness, not pretend to be real time. Show coordinators when status was last checked and distinguish submitted from delivered. The honest label is more useful than a green badge backed only by request acceptance.

## How should suppressions and report email interact?

They should share a campaign record, not transport assumptions. The record can say that tournament report `R-1042` produced one email job and a volunteer-alert cohort; the email adapter and SMS adapter then proceed independently. That preserves an audit trail without coupling attachment delivery to carrier state.

Suppressions are a transport guard, not the nonprofit's whole consent model. Keep the reason a volunteer may be contacted, the time permission changed, the intended message class, and the selected destination. Resolve duplicates and suppressions before a batch. Keep broader outreach separate from an urgent shift cancellation even when both happen to use SMS.

The email lane has its own edges. There is no SMTP relay or hosted email OTP operation in this surface, and scheduled email has no cancellation route. SMS does have cancellation. Do not infer symmetry between channels. Make the generated attachment final before scheduling its email, and evaluate Resend independently if its email-focused integration better fits that lane. The FTC's CAN-SPAM guide is relevant to commercial email compliance, but it does not decide SMS consent or regional sender rules.

**Decision rule:** choose the consolidated REST option when the team values one key, one bill, discoverable contracts, and can tolerate queued polling. Choose Twilio or Vonage when callback behavior or richer communications channels drives the architecture. Choose Amazon SNS when the existing AWS boundary removes more integration work than its topic model adds. Keep the emailed gaming report as a separate decision in every case.

## Sources

- [Twilio: Track the Message Status of Outbound Messages](https://www.twilio.com/docs/messaging/guides/track-outbound-message-status)
- [Vonage SMS API overview](https://developer.vonage.com/en/messaging/sms/overview)
- [Amazon SNS: Sending SMS messages](https://docs.aws.amazon.com/sns/latest/dg/sms_publish-to-phone.html)
- [Resend documentation](https://resend.com/docs/introduction)
- [FTC: CAN-SPAM Act compliance guide](https://www.ftc.gov/business-guidance/resources/can-spam-act-compliance-guide-business)
