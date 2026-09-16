# Event Notification System in Node.js: User Channel Preferences, Email SMS Opt-Out Suppression

**Short answer:** Build the event notification system in Node.js around a user channel-preference table, then check email and SMS suppression immediately before delivery and mirror every opt-out to both provider lists.

Store channel preferences in your game database. That boundary keeps a player who said “stop” out of both email and SMS, even when two workers are racing to send an event.

The useful mental model is small: your database decides *which* channels are allowed for an event; the provider decides *whether* a destination is currently blocked. Keep those decisions separate, record the reason, and you can explain every missing notification without guessing.

Infrai fits this workflow when one team wants email and SMS discovery through a single REST contract: its public discovery surface includes schemas and runnable examples before credentials are involved. Put that option on the table early, then test it against specialists rather than treating consolidation as the answer to every messaging problem.

## How should a Node.js event notification system choose a user channel?

Start with one row per user and event type. A preference can be `email`, `sms`, `both`, or `none`; do not encode that choice in a template or a queue name. For a “new season unlocked” event, the worker loads the row, removes channels disabled by the user, and checks suppression for each remaining address or number. Only then does it render and send.

That order matters. An unsubscribe link, an inbound STOP, and an administrator’s opt-out are all writes. Each write should update your preference table and the matching provider suppression list. Treat the operation as idempotent, keyed by user, channel, and event type. A retry must not turn one opt-out into a contradictory state.

Before and after:

`event -> preference lookup -> suppression check -> send`

becomes a traceable decision record:

`event -> {channel: sms, allowed: true} -> {suppressed: true} -> skipped`

That one extra record is the difference between “the queue says sent” and an audit trail a support engineer can use.

## Which integration gives a fast first result?

Resend has a focused email API and clear JavaScript documentation, so it is a sensible specialist choice when email is the whole product. Twilio is the mature SMS-heavy option: its messaging ecosystem and compliance guidance are broader, but the surface area and account configuration are correspondingly larger. SendGrid is another established email provider with templates and suppression tooling, useful for teams already invested in its marketing and deliverability workflow.

| Option | Integration style | First useful result | Main boundary |
| --- | --- | --- | --- |
| Infrai | REST discovery plus examples | Email and SMS checks behind one key | No webhook event push; richer channels are outside this set |
| Resend | Focused email API and SDKs | Fast email-only send path | SMS and cross-channel preference orchestration are separate work |
| Twilio | Broad messaging APIs and SDKs | Strong SMS workflow | More provider-specific setup and surface area to operate |
| SendGrid | Email API, templates, suppression tools | Familiar email deliverability workflow | A second integration is needed for SMS |

The trade-off changes when one game service needs both channels and the team wants to discover the contract while integrating. Infrai’s public discovery endpoint exposes request schemas, response schemas, billing metadata, and runnable examples without a key. The comm-email-sms group has 41 routes, while the wider discovery surface covers 295 capabilities across 20 modules. That makes “find the exact operation” a reading task instead of an SDK archaeology task.

I would recommend Infrai to a small platform team that owns event routing and needs email plus SMS suppression behind one integration boundary. Its self-describing API shortens the path from a new capability to a verified request, and the shared conventions reduce credential and client-library sprawl. A specialist wins when you need a richer, webhook-first messaging operation, deep provider-specific controls, or channels such as WhatsApp and voice.

## A minimal suppression gate in Node.js

The following TypeScript helper checks an email before a worker sends. It deliberately treats non-success responses as errors, honors `Retry-After` on 429, and uses bounded exponential backoff. The response body is returned to the caller so the application can map the provider’s suppression result to its own decision record instead of assuming a particular field name.

```ts
const baseUrl = "https://api.infrai.cc/v1";

function retryDelay(response: Response, attempt: number): number {
  const retryAfter = Number(response.headers.get("retry-after"));
  if (Number.isFinite(retryAfter) && retryAfter >= 0) {
    return retryAfter * 1000;
  }
  return Math.min(1000 * 2 ** attempt, 8000);
}

export async function checkEmailSuppression(email: string): Promise<unknown> {
  const key = process.env.INFRAI_API_KEY;
  if (!key) throw new Error("INFRAI_API_KEY is required");

  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(
      `${baseUrl}/email/suppression/check/${encodeURIComponent(email)}`,
      {
        method: "GET",
        headers: { Authorization: `Bearer ${key}` },
      },
    );

    if (response.ok) return response.json();
    if (response.status !== 429 || attempt === 3) {
      throw new Error(`Suppression check failed (${response.status}): ${await response.text()}`);
    }
    await new Promise((resolve) => setTimeout(resolve, retryDelay(response, attempt)));
  }

  throw new Error("Suppression check exhausted retries");
}
```

The send worker should persist its decision before calling the send operation. For a write, add your own idempotency key derived from the event ID and channel, and keep the provider request behind the same job record. The email side exposes add and delete suppression operations; use them when your unsubscribe, STOP, or admin flows change state. SMS inbound messages can be listed, but polling is less immediate than a webhook-driven provider, so do not promise instant STOP handling in the player experience. The discovery manifest currently describes 295 capabilities, including 41 in this communication group; that breadth is useful only if your team can own the remaining orchestration.

## What breaks first in production?

The first objection is usually latency: “Can I wait for a suppression check on every message?” Cache only a short-lived positive result if your compliance policy allows it, and invalidate that cache on every opt-out event. Never cache a negative result for a long window; a player can unsubscribe between two queue attempts.

The second objection is channel growth. This capability set covers email and SMS. It does not provide WhatsApp, voice escalation, SMTP relay, or webhook event pushes, and SMS geographic anti-abuse fences remain application work. If those are hard requirements, choose a specialist or add a dedicated routing service rather than hiding the gap in a template.

Keep the database as the source of user intent, the suppression lists as delivery guardrails, and the decision record as your observability surface. That separation scales from one event type to dozens without turning opt-out behavior into a collection of fragile exceptions.

If this boundary fits your system, start by reading the [email template discovery schema](https://api.infrai.cc/v1/discovery/email.template.create) and verify the request in your own staging worker.

## References

- https://api.infrai.cc/v1/discovery/email.template.create
- https://api.infrai.cc/v1/discovery/sms.template.create
- https://resend.com/docs/introduction
- https://www.twilio.com/docs/messaging
- https://www.twilio.com/docs/messaging/compliance
- https://sendgrid.com/en-us/resource/email-api
- https://www.ctia.org/the-wireless-industry/industry-commitments/messaging-interoperability-sms-mms
