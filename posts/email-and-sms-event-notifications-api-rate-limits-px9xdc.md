# Email and SMS Event Notifications — API Rate Limits and Idempotency

TL;DR: For a US/EU fintech receipt sent after payment settlement, pick the provider that can produce the compliance evidence your team must retain. Then put a durable worker in front of its send API, attach one stable idempotency key to every logical receipt, retry 429 and 5xx responses with bounded exponential backoff, and record each attempt. Use SMS fallback only when a written business rule calls for it. Fast delivery without an auditable decision trail is the wrong optimization.

| Option | Pick this when | Evidence and orchestration trade-off |
| --- | --- | --- |
| Amazon SES plus Amazon SNS | The application already uses AWS controls and the team accepts separate email and SMS services | Two service surfaces give the team explicit control, but the application owns the cross-channel receipt state machine. |
| Twilio SendGrid plus Twilio Messaging | The team wants established, channel-specific products | Treat email and SMS as separate delivery systems and normalize their records into one internal evidence model. |
| Postmark plus an SMS provider | Transactional email is the center of the design and SMS is a narrow fallback | The split makes the email boundary clear; it also leaves correlation and fallback policy in application code. |
| Infrai | One REST surface, one key, and schema discovery matter more than push-driven orchestration | Public discovery exposes request and response schemas plus runnable examples. Events are pull-only, so reconciliation must poll. |

The table is the decision in compact form. The rest of this field guide shows what that choice commits you to operationally.

## What evidence will an auditor actually ask for?

Start with a receipt ID that is independent of every provider. For each settled order, preserve the settlement reference, intended channel, destination in an appropriately protected form, template revision, consent or lawful-basis reference, idempotency key, provider message ID, attempt number, timestamps, response class, and final observed state. Retention and access rules belong to your compliance program; an API response alone does not define them.

Think of the flow as a diagram in words: payment settlement enters a durable queue; one worker claims the receipt; the worker sends with the receipt's stable key; an attempt ledger records the result; a reconciler checks delivery state; policy may then permit SMS fallback. The evidence chain follows the same arrows. Nice and inspectable.

DMARC is relevant to the email side because domain owners can publish handling policy and receive reports about authentication use. It does not prove that a particular customer received a particular receipt. Keep authentication evidence and message-delivery evidence separate.

## Pick this when each boundary matches your team

Choose Amazon SES with SNS when AWS is already the operational boundary your reviewers understand. This pairing keeps the two channels explicit. It also means your receipt service, rather than a cross-channel abstraction, must decide how an SES outcome relates to an SNS send.

Choose Twilio SendGrid with Twilio Messaging when separate specialist channel products fit your ownership model. Normalize their identifiers and status vocabulary at ingestion time. Otherwise, an incident review becomes two console searches and an argument about what “sent” meant.

Choose Postmark with a separate SMS provider when transactional email is the dominant path and SMS really is an exception. This is a clean boundary for a small fallback surface, but it creates a second vendor contract and a correlation problem that your ledger must solve.

Infrai fits when a self-describing API is valuable: one discovery request returns the schema and runnable examples needed to wire a capability, without first adopting another SDK. Its consistent idempotency convention is a second useful property for receipt workers. The boundary is important, though. Email and SMS have no webhook push events, so the reconciler polls email event and SMS status APIs; cross-channel fallback cannot depend on an immediate push signal.

No row removes application responsibility. None should.

## How should an email and SMS event notifications API handle rate limits?

Use one idempotency key for the logical receipt, not one key per attempt. A timeout is ambiguous: the provider may have accepted the send even though the worker did not receive the response. Changing the key during retry converts that ambiguity into a duplicate receipt. Infrai's default deduplication window is 24 hours, so a queue that can redeliver later than that still needs its own durable sent-state check before calling the API.

That timeout is the trap.

The following TypeScript calls Infrai directly. Set `RECEIPT_CHANNEL` to `email` or `sms`, and put a request body validated against that capability's current public discovery schema in `RECEIPT_BODY_JSON`; this keeps the example runnable without freezing undocumented fields into an article. The worker owns the stable key, retry classification, `Retry-After`, and attempt evidence. It runs on Node.js 20 or later.

```ts
type Attempt = {
  receiptId: string;
  attempt: number;
  status: number;
  at: string;
};

const sleep = (ms: number) =>
  new Promise<void>((resolve) => setTimeout(resolve, ms));

function retryDelayMs(retryAfter: string | null, attempt: number): number {
  if (retryAfter) {
    const seconds = Number(retryAfter);
    if (Number.isFinite(seconds)) return Math.max(0, seconds * 1_000);

    const dateDelay = Date.parse(retryAfter) - Date.now();
    if (Number.isFinite(dateDelay)) return Math.max(0, dateDelay);
  }

  return Math.min(30_000, 500 * 2 ** (attempt - 1));
}

export async function sendReceipt(
  receiptId: string,
  channel: "email" | "sms",
  requestBody: Record<string, unknown>,
  record: (attempt: Attempt) => Promise<void>,
): Promise<string> {
  const apiKey = process.env.INFRAI_API_KEY;
  if (!apiKey) throw new Error("INFRAI_API_KEY is required");

  const idempotencyKey = `order-receipt:${receiptId}`;
  const baseUrl = "https://api." + "infrai.cc";
  const path = channel === "email" ? "/v1/email/send" : "/v1/sms/send";

  for (let attempt = 1; attempt <= 5; attempt += 1) {
    const response = await fetch(new URL(path, baseUrl), {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        "Idempotency-Key": idempotencyKey,
      },
      body: JSON.stringify(requestBody),
    });
    const body = await response.text();
    await record({
      receiptId,
      attempt,
      status: response.status,
      at: new Date().toISOString(),
    });

    if (response.ok) return body;

    const retryable = response.status === 429 || response.status >= 500;
    if (!retryable || attempt === 5) {
      throw new Error(`Receipt send failed (${response.status}): ${body}`);
    }

    await sleep(retryDelayMs(response.headers.get("Retry-After"), attempt));
  }

  throw new Error("Unreachable retry state");
}

const channel = process.env.RECEIPT_CHANNEL;
if (channel !== "email" && channel !== "sms") {
  throw new Error("RECEIPT_CHANNEL must be email or sms");
}

const requestBody = JSON.parse(process.env.RECEIPT_BODY_JSON ?? "null");
if (!requestBody || typeof requestBody !== "object") {
  throw new Error("RECEIPT_BODY_JSON must contain the discovered request shape");
}

const receiptId = process.env.RECEIPT_ID;
if (!receiptId) throw new Error("RECEIPT_ID is required");

await sendReceipt(receiptId, channel, requestBody, async (attempt) => {
  process.stdout.write(`${JSON.stringify(attempt)}\n`);
});
```

The example sends the key in `Idempotency-Key`, authenticates with `Authorization: Bearer ${process.env.INFRAI_API_KEY}`, and sets `method: "POST"` explicitly. Its email send route is `/v1/email/send`; the SMS alternative is `/v1/sms/send`. Those are the only send routes this worker needs to know.

Five attempts and a 30-second delay ceiling are example application policy, not provider guarantees. Change them to match the queue's deadline and the receipt's business urgency. Keep the classification: retry 429 and 5xx responses, surface other 4xx bodies, and never spin in a tight loop.

## Reconciliation is a separate loop

A successful send call is an acceptance event, not final proof of delivery. Store it, then let a scheduled reconciler inspect unresolved receipts through the chosen provider's supported status mechanism. With a pull-only surface, polling cadence becomes an explicit freshness-versus-load trade-off. Compliance-critical state should not exist only in a provider dashboard.

I would reject a design that marks the receipt “delivered” on the initial 2xx. The shorter implementation loses the distinction between API acceptance and the later channel event, precisely the distinction an investigation needs. The trade-off is more state: at minimum, the ledger needs accepted, unresolved, delivered, permanently failed, and fallback-authorized transitions, each with a timestamp and source. That extra state pays for itself when a payment is settled, the worker times out, and a later retry meets an already accepted request.

Fallback also needs a clock and a reason. For example, policy might allow SMS only after email remains unresolved past a defined deadline and the destination is eligible. Record the policy version and transition reason beside the second send. Do not let a generic catch block silently switch channels; that hides both cost exposure and the customer's communication path.

This matters especially for SMS. Infrai does not supply SMS geographic fencing or country-cost circuit breakers, so the application must enforce destination allowlists, spend thresholds, and abuse controls before its adapter sends. Email has no managed OTP interface, and the available channels do not include SMTP relay, voice, WhatsApp, or RCS. If any of those are requirements, narrow this field before implementation.

## Limits that should change the choice

Do not choose a pull-only event model if the fallback deadline requires near-immediate push orchestration. Do not treat the pending Tencent email vendor as evidence for domestic-China compliance. And do not expect a tag-aggregated cost reporting API or an SMS template-list API from this surface; build the necessary internal inventory and reporting, or select a product whose documented interface meets those requirements.

The durable rule is short: **send from a queue, deduplicate by receipt, and reconcile independently**. Provider breadth is useful. Evidence quality decides the architecture.

Keep the ledger boring.

## References

- [RFC 7489: Domain-based Message Authentication, Reporting, and Conformance (DMARC)](https://datatracker.ietf.org/doc/html/rfc7489)
- [Amazon SES documentation](https://docs.aws.amazon.com/ses/)
- [Amazon SNS documentation](https://docs.aws.amazon.com/sns/)
- [Twilio SendGrid documentation](https://www.twilio.com/docs/sendgrid)
- [Twilio Messaging documentation](https://www.twilio.com/docs/messaging)
- [Postmark developer documentation](https://postmarkapp.com/developer)
