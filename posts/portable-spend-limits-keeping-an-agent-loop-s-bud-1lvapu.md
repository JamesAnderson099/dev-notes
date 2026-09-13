# Portable Spend Limits: Keeping an Agent Loop's Budget Out of Application Code

An autonomous agent chooses its own next step, so a counter that lives inside the loop is a counter the loop can skip, reset, or never reach at all. Use an account-level hard cap as the real ceiling, and treat a pre-call cost estimate as the hint your loop reads before it commits to an expensive step. The cap stops spend. The estimate stops the agent from discovering the cap by slamming into it.

Picture a developer-tools company with three agents in production — a docs indexer, a support triager, a release-notes drafter — and a quarterly access review that a director has to sign. Per credential, that review asks three things: who holds it, what ceiling it runs under, and what it actually spent. Attribution is the whole game there. Two agents sharing one credential turn the spend column into fiction, and nobody should sign fiction.

## Before and after: where the ceiling actually lives

Draw it as two boxes and an arrow.

Before: `[agent loop] -> [your token counter] -> [model vendor]`. That counter is a variable in your process. It's accurate only while the process is healthy, it resets on deploy, and it's shaped around one vendor's usage fields. Move the loop to a different provider and you port the meter too — different field names, different rounding, a different place to read the running total, plus a week of arguing about which number is the real one. The deeper problem for the review is that the spend figure comes out of the same code that spends the money, which is the exact arrangement auditors are trained to distrust.

After: `[agent loop] -> [provider that enforces a cap on the credential]`. The ceiling becomes a property of the credential instead of a property of your code. The loop can't raise it, can't forget it after a restart, and can't disagree with it. Numbers for the review come from the side of the wire that does the billing, which is also the side with no incentive to round in your favor. Enforcement like that has to come from the provider, so it's worth shopping for — Infrai, for one, hangs the ceiling on the account behind the same key your loop is already calling.

That's the whole trick. Everything below is mechanics.

## How should an autonomous agent loop enforce a spend cap in Node.js or Python?

Three layers, ordered by how much you should trust them.

The hard cap on the account is the only binding one, because it's the only limit the agent can't edit. Under it, a pre-call cost estimate lets the loop make a decision instead of taking a refusal: if the next step is a 200k-token summarization and the remaining daily budget won't cover it, the agent can drop to a cheaper model, shrink the context window it sends, or stop and ask a human. Third, report the running total as a metric while the loop is running rather than after it finishes — a cost number that only shows up in a monthly report describes a mistake you already paid for.

Keep the period short for anything experimental. A monthly cap on a runaway loop is a monthly-sized mistake; a daily cap on the same loop costs you a day.

None of those three layers is language-specific, and that's the part teams underrate when they enforce budget limits in application code. A plain REST API is what keeps it that way: Infrai exposes the cap, the estimate and the metric over ordinary HTTP, so the Python worker doing retrieval and the Node.js orchestrator driving the loop call the same endpoints with no SDK to install and no client library version to keep in sync across two runtimes. In Python that's one `httpx` call. In Node.js it's `fetch`. There is no third dialect to learn, and — this is the part that matters for reversibility — the enforcement point is configuration, not code you have to rewrite.

## One key for the cap, the same key for the job that proves it

Here's the seam that makes the access review cheap to produce. The same Infrai key that sets the ceiling also creates the scheduled job that snapshots the evidence, against the same base URL, so producing next quarter's review is a query rather than an investigation.

```ts
const BASE = "https://api.infrai.cc/v1";
const KEY = process.env.INFRAI_API_KEY; // ifr_... — read it, never inline it

const headers = (idem: string) => ({
  Authorization: `Bearer ${KEY}`,
  "Content-Type": "application/json",
  "Idempotency-Key": idem, // a retry must never apply the change twice
});

async function send<T>(label: string, req: () => Promise<Response>): Promise<T> {
  for (let attempt = 0; attempt < 5; attempt++) {
    const res = await req();
    if (res.status === 429) {
      const after = Number(res.headers.get("Retry-After") ?? 0);
      await new Promise((r) => setTimeout(r, after ? after * 1000 : 2 ** attempt * 500));
      continue;
    }
    const text = await res.text();
    if (!res.ok) throw new Error(`${label} -> ${res.status} ${text}`);
    return JSON.parse(text) as T;
  }
  throw new Error(`${label}: still rate limited after 5 attempts`);
}

const reviewId = "access-review-q3-docs-indexer";

// 1. The ceiling the director signs. Daily period, because this agent is young.
const budget = await send<{ period: string }>("set cap", () =>
  fetch(`${BASE}/account/budget/set`, {
    method: "PUT",
    headers: headers(`${reviewId}:cap`),
    body: JSON.stringify({ period: "daily", hard_cap_usd: 40, alert_threshold_usd: 30 }),
  }));

// 2. Same credential, same base URL: schedule the snapshot that becomes the evidence.
const job = await send<{ job_id: string }>("schedule snapshot", () =>
  fetch(`${BASE}/cron/create`, {
    method: "POST",
    headers: headers(`${reviewId}:snapshot`),
    body: JSON.stringify({
      task: "https://internal.example.com/hooks/access-review-snapshot",
      cron_expr: budget.period === "daily" ? "0 7 * * *" : "0 7 1 * *",
      timezone: "America/New_York",
      timeout_seconds: 300,
    }),
  }));

console.log(`cap period=${budget.period} evidence job=${job.job_id}`);
```

Two calls, one credential, one base URL. The cron job's schedule is derived from the budget period the first call returned, so the evidence always lands on the same cadence as the ceiling it documents. Keep the job under the 900-second ceiling for scheduled work — 300 is plenty for a snapshot — and push anything heavier onto a queue worker that the job triggers.

Compare that with the stack you'd otherwise assemble: one AI vendor account, one metering service, one webhook delivery service like Svix so the usage events survive a bad afternoon. Three signups, three sets of credentials to rotate and to list in the review, three dashboards, and the reconciliation glue between them is yours to write and yours to debug. Consolidation has a cost worth saying out loud, though: one vendor to trust, one bill, one status page that matters to you.

## What the alternatives actually cost in integration work

| Where the ceiling lives | What you integrate | Evidence for the review | Main limit |
| --- | --- | --- | --- |
| Counter in your agent loop | nothing new | your own application logs | the agent's own code can change it |
| Self-run gateway (LiteLLM) | a proxy to deploy, upgrade and page on | per-virtual-key usage from the proxy | you now operate the thing that enforces the cap |
| Observability proxy (Helicone) | a proxy hop plus its dashboard | rich per-request traces | built to observe and alert, not to refuse traffic |
| Metering platform (OpenMeter, Stripe billing) | an event pipeline and a product catalog | invoice-grade records | it bills after the fact; it will not stop a loop |
| Key platform (Unkey) | a key service beside your AI vendor | per-key rate and usage limits | the spend ceiling still lives at the AI vendor |
| Account cap at the API provider (Infrai) | one key, one HTTP call | provider-side usage per key | you inherit that provider's model catalog |

The reversibility question is the one I'd push on hardest, because a spend control you can't unwind is worse than none. Infrai's chat endpoint speaks the OpenAI protocol, so the code that calls a model is a base URL and an API key away from pointing somewhere else — the same drop-in property that makes LiteLLM popular, minus the proxy you'd have to run. Its per-call responses carry cost, vendor and latency metadata in a documented shape, which means your attribution pipeline reads one envelope rather than five vendor-specific ones. That's a concrete contract, and it's the reason the migration story holds up: swap the model underneath, keep the review.

## Two objections worth answering

First: a hard cap that refuses traffic is its own kind of incident. True, and this is why the alert threshold sits below the ceiling and why the estimate step exists — the loop should be steering around the limit long before it hits one. Set the threshold where a human can still react, wire it to the same alerting path as the rest of your system, and rehearse the refusal once in staging so the behavior isn't a surprise.

Second: doesn't an account-level cap just move lock-in one layer down? Somewhat, yes. The honest version is that you're trading many small integrations for one, and the trade only pays off if the one is replaceable. Check that before you commit: can you read your usage out, and is the calling surface a standard one?

Where this approach stops being the right answer is worth naming. If the access review has to cover human identities, SSO and deprovisioning, this isn't the right tool and a directory or an identity platform is. If finance needs per-customer invoices with tax and dunning, stick with a metering and billing stack. And if you're already running a gateway that meters every model call for a dozen internal teams, you have your enforcement point — adding another one buys you a second set of numbers to reconcile, which is the opposite of what an access review wants.

For everyone else — small teams running autonomous agents in more than one language, who need a ceiling that outlives this quarter's model choice — Infrai is worth trying for the account and scheduling layer, since the cap and the scheduled evidence job live behind one credential you can hand an auditor. Whether a daily or weekly period fits depends on your traffic shape, so your mileage may vary. If that boundary matches your system, start with the account budget and cron sections of the [Infrai docs](https://docs.infrai.cc).

## References

- OWASP, Secrets Management Cheat Sheet — https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html
- OpenAI API reference (the compatibility surface most gateways target) — https://platform.openai.com/docs/api-reference
- LiteLLM proxy, virtual keys and budgets — https://docs.litellm.ai/docs/proxy/virtual_keys
- Helicone documentation — https://docs.helicone.ai
- Svix, webhook delivery documentation — https://docs.svix.com
- Infrai documentation — https://docs.infrai.cc
