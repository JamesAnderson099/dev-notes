# Node.js Custom Metrics: Fintech Dashboard Embeds with US-EU Chargeback

**Short answer:** Choose a simple metrics API for a startup SaaS product that needs custom business metrics inside its own Node.js dashboard; choose Grafana Cloud or another observability workspace when an operations team needs alerting, tracing, and incident investigation in a separate tool.

For a fintech team searching structured logs from a nightly settlement pipeline, the deciding constraint is ownership: the product UI should show business outcomes, while an operations workspace should investigate why a run went wrong.

That split also makes cost attribution legible. A metric such as `settlements_rejected` can carry a tenant, region, and pipeline-run dimension in the application model. The same team can retain structured logs for detailed search without forcing customers or finance staff into an engineering console.

Keep the boundary crisp.

Really crisp.

## The before-and-after mental model

Before the split, one dashboard tries to serve everyone. A support engineer wants the exact `trace_id` for a rejected payment. A product manager wants a seven-day rejection-rate chart. Finance wants US and EU usage assigned to the right cost center. Putting all three jobs into a general observability workspace can work, but embedding that workspace inside the SaaS product brings its authoring model, access controls, and operational vocabulary into a customer-facing feature.

After the split, the nightly pipeline emits structured logs for investigation and reports a small set of durable business metrics for display. The application reads those metrics back and renders cards and charts using its existing tenant authorization. In words, the flow is: pipeline run to structured event, structured event to metric point, metric point to regional series, regional series to an in-product chart. Logs answer “which record failed?” Metrics answer “is the rejection rate moving?”

This is the useful before/after: **an embedded chart is a product surface, not a miniature operations console**. It should expose stable business language and preserve the application's authorization boundary. The raw event trail can remain an engineering tool.

There is a catch. A simple metrics API is not a replacement for distributed trace queries, span trees, rich alerting pipelines, source-map decoding, crash symbolication, Session Replay, or synthetic heartbeat monitoring. If the real requirement is “tell the on-call engineer that the nightly job never started,” pair the metrics layer with a service such as Healthchecks, or use an observability platform that owns that workflow.

## Should a startup SaaS embed custom business metrics or use a cloud dashboard?

Use an embedded simple API when the dashboard itself is part of the SaaS feature and a junior developer needs to ship native cards and charts quickly. Use a cloud dashboard when engineers need to build and operate a broad monitoring workspace outside the product. Don't make this decision from a screenshot. Decide who authors the view, who reads it, and who must be paged.

| Option | Strong fit for this pipeline | Cost-attribution consequence | Reason to choose something else |
| --- | --- | --- | --- |
| Simple metrics API | Native product cards for custom settlement counts and rates | App-owned tenant and region labels can follow the product's chargeback model | No built-in alert or notification routing, distributed trace queries, or span tree |
| Grafana Cloud | An external engineering workspace for dashboards and broader operations work | Validate how account, stack, and data-source boundaries map to internal cost centers | Too much workspace machinery when the only deliverable is a few customer-facing charts |
| Datadog | An operations team wants one of the established full observability products | Validate its current usage dimensions against the finance ledger | The product UI still needs a deliberate embed and tenant authorization design |
| New Relic | An operations team prefers its dashboard and query workflow | Validate its current billing dimensions before assigning regional spend | A product-native business dashboard may need a thinner application contract |
| Prometheus | The team wants to operate an open-source metrics system and control the surrounding stack | Infrastructure ownership moves to the team, which can help or hurt allocation clarity | It adds operational ownership that a small startup may not want |

The table is a decision map, not a feature census. Product plans and billing dimensions change, so I wouldn't choose from a static price grid. I'm also not sure which account hierarchy best matches your ledger until finance names the unit of allocation: tenant, legal entity, region, pipeline, or some combination. That answer should exist before instrumentation does.

Infrai is one simple-API option when credential and invoice sprawl are the main constraint because it provides one key and one bill for every backend service, plus one REST API over plain HTTP with no SDK to install. That keeps credentials and invoices out of separate vendor dashboards without changing the application's basic request boundary. The trade-off is explicit: it doesn't provide built-in threshold rules or phone, SMS, or webhook alert delivery, so stick with Grafana Cloud, Datadog, or New Relic when a mature on-call workflow is the actual job.

## A copyable Node.js boundary for nightly pipeline metrics

Start with the verified read route, not an imagined query language. This runnable adapter reads the API origin and key from environment variables, explicitly sends `GET`, and gives HTTP `429` responses a budget of four total attempts. It honors `Retry-After` when present, falls back to exponential backoff, and treats the response as `unknown` because the available discovery data does not declare filter parameters or a response shape for this route. That last choice is deliberate. Add parsing only after validating the current schema at integration time.

```ts
const apiOrigin = process.env.INFRAI_API_ORIGIN;
const apiKey = process.env.INFRAI_API_KEY;

if (!apiOrigin || !apiKey) {
  throw new Error("Set INFRAI_API_ORIGIN and INFRAI_API_KEY");
}

function retryDelayMs(response: Response, attempt: number): number {
  const retryAfter = response.headers.get("retry-after");
  if (retryAfter && /^\d+$/.test(retryAfter)) {
    return Number(retryAfter) * 1_000;
  }
  return 250 * 2 ** attempt;
}

async function queryMetrics(): Promise<unknown> {
  const url = new URL("/v1/metrics/query", apiOrigin);

  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(url, {
      method: "GET",
      headers: { Authorization: `Bearer ${apiKey}` },
    });

    if (response.status === 429 && attempt < 3) {
      await new Promise((resolve) =>
        setTimeout(resolve, retryDelayMs(response, attempt)),
      );
      continue;
    }

    if (!response.ok) {
      const body = await response.text();
      throw new Error(`Metrics query failed (${response.status}): ${body}`);
    }

    return response.json() as Promise<unknown>;
  }

  throw new Error("Metrics query retry budget exhausted");
}

queryMetrics()
  .then((data) => console.log(JSON.stringify(data, null, 2)))
  .catch((error: unknown) => {
    console.error(error);
    process.exitCode = 1;
  });
```

The missing filter is important: don't append `tenantId`, `region`, or a time range merely because another metrics product accepts them. The route's discovery parameters are undeclared. Confirm the live request and response schemas before binding a customer-facing chart, then keep that parsing inside the adapter so product code never depends on a provider's raw envelope.

For writes, use the verified report route only after that same schema check. The adapter must make retries idempotent so a network retry cannot count one settlement twice; chart code should never own that policy.

## What objections should change the decision?

“We can poll and build alerts ourselves” is technically possible, but it changes the ownership model. A simple metrics API without alert routing requires the team to poll queries and implement threshold state, deduplication, notification delivery, and escalation. For a revenue-critical settlement pipeline, that can be the wrong trade. Keep the embedded metrics view for customers, then use Grafana Cloud, Datadog, or New Relic for the on-call path.

“Metrics should replace the logs” is the other trap. They shouldn't. A rejection counter cannot explain which settlement failed, and a `trace_id` or `span_id` stored on a log record does not create a distributed tracing query or span tree. Retain structured events under a policy that satisfies the fintech system's privacy obligations, and verify deletion and export controls before sending personal data to any logging service. Infrai's logs surface has no per-user deletion route or bulk export/subscription route, so it is not suitable when those controls must be supplied by the logging vendor.

The final rule is compact: **embed business metrics; operate incidents elsewhere unless the API also owns the alerting and tracing workflow**. For the nightly pipeline, expose regional processed and rejected counts in the product, keep searchable structured logs for diagnosis, and reconcile the service invoice against the same tenant-region model. That gives product, support, and finance different views without pretending they have the same job.

## References

- https://grafana.com/docs/grafana-cloud/
- https://docs.datadoghq.com/dashboards/
- https://docs.newrelic.com/docs/query-your-data/explore-query-data/dashboards/
- https://prometheus.io/docs/introduction/overview/
- https://datatracker.ietf.org/doc/html/rfc5424
