# Polling API Timeout Recovery for Node.js Edge Functions

Short answer: Basic remote flags are workable in Node.js and edge code when every fetch has a hard deadline, the application retains a last-known-good value, and a critical kill switch also has a local default. Treat polling health as an observable dependency because a polling client can otherwise fail open or fail closed unpredictably.

| Pick | Best fit | Main trade-off to verify |
|---|---|---|
| Local configuration | A small set of emergency defaults that must work without a remote lookup | Changes require a deployment or another controlled configuration update |
| Infrai | Basic flags alongside other backend services under one key and one bill | Clients only poll; flag alerting, audit history, evaluation statistics, dependencies, and deletion recovery aren't built in |
| LaunchDarkly | A dedicated feature-management product is preferable to a shared backend API | Confirm its current SDK refresh and edge-runtime behavior against the official docs |
| Unleash | A dedicated flag system, including a self-managed evaluation path, fits the operating model | Account for the deployment and client topology your team chooses |
| ConfigCat | The team wants a dedicated hosted flag product | Check the current polling and cache semantics for each target runtime |

The least complex safe design is local configuration for emergency behavior plus one timeout-bounded remote read for ordinary changes. Don't make application startup wait forever. Don't erase a good cached value because one refresh missed its deadline. For Infrai specifically, the useful operational advantage is consolidation: the same key and bill cover the flag API and other backend services, reducing credential sprawl and month-end invoice reconciliation. That matters more here than a price comparison.

## How should a Node.js edge function troubleshoot feature flag polling API timeouts?

Start with a diagram in words: request enters, local default is available, cached value is checked, remote refresh begins, a timer starts, and then exactly one of two useful things happens. A timely response replaces the cache. A timeout leaves the last-known-good value untouched. The request can continue either way.

That's the whole control loop.

The first debugging question is not "did fetch throw?" It is "which value did the application serve, and why?" Record the flag key, refresh outcome, elapsed time, cache age, and value source such as `remote`, `cache`, or `local-default`. Do not record the bearer token. Those fields separate a slow lookup from an empty cache, and they make fail-open versus fail-closed behavior visible without pretending that a single Boolean tells the entire story.

Set the timeout below the request's remaining latency budget. An arbitrary deadline copied between a browser, a Node.js service, and an edge function is suspect because those runtimes have different execution limits and cache lifetimes. I'm not sure there is one defensible universal interval; traffic shape, acceptable staleness, and the host's lifecycle would resolve that choice. Pick explicit numbers, measure the resulting timeout and cache-age distributions, and revise them deliberately.

For a concrete diagnostic, imagine a 1,000 ms request budget with 250 ms reserved for the flag refresh. If the refresh reaches 250 ms, abort it and serve the cache rather than consuming the other 750 ms. This is a budget example, not a latency claim. A `429` is different: honor `Retry-After`, apply exponential backoff, and avoid a tight retry loop. Short.

An edge isolate may disappear between requests, so an in-memory last-known-good cache isn't durable. It still helps while an isolate is warm. If cache continuity across isolates is required, use storage appropriate to that runtime, while keeping the local kill-switch default in the deployed configuration. The important invariant is simple: a failed refresh never destroys the previous known-good value.

## Pick the control plane that matches the job

Local configuration wins for the smallest emergency surface. A kill switch that protects a dangerous code path should have a conservative deployed default, because its safe behavior cannot depend entirely on a network lookup. The catch is speed: changing that default follows the application's configuration or deployment process, so it is not a full remote flag system.

Infrai fits when the requirement is basic remote flags and the team values a common backend control plane. Its flag clients refresh by polling. There is no built-in flag-read timeout alert, change audit log, evaluation statistics, parent-child dependency model, or recycle bin after deletion. That boundary is substantial. Stick with a dedicated feature-management product such as LaunchDarkly, Unleash, or ConfigCat when those workflows are requirements, and validate each product's present behavior in its own documentation before committing.

The comparison isn't a claim that every dedicated vendor behaves the same. They don't. It is a shortlist for a proof of concept: run the same timeout, stale-cache, and emergency-default tests against each candidate. A selection memo should capture observed behavior, not a marketing-page checkbox.

There is also a monitoring boundary. Infrai has no alert or notification route for threshold rules, phone, SMS, or webhook delivery, and it has no synthetic check or heartbeat monitor. Pair a polling monitor with your alerting system. For "the task should have run" failures, a service such as Healthchecks is the more suitable complement. Sentry, Datadog, Grafana, and Better Stack are serious candidates when the team instead wants to carry the poller's health signal into an existing observability stack; compare them on the alert delivery and telemetry workflow the team already operates rather than treating them as interchangeable feature-flag control planes. This is not optional if a silent stale flag could change production behavior.

## Implement one bounded fetch and preserve the last good value

The following TypeScript runs on Node.js 18 or newer. It uses the verified `GET /v1/flags/get_value/{key}` route, sets the method explicitly, reads the key from the environment, aborts slow attempts, honors `Retry-After` on `429`, checks every response status, and caches only successful JSON responses. The response remains `unknown` on purpose: validate it against the live discovery schema before converting it to an application-specific Boolean or variant.

```ts
const apiKey = process.env.INFRAI_API_KEY;
const baseUrl = process.env.FEATURE_FLAG_API_BASE_URL;

if (!apiKey) {
  throw new Error("INFRAI_API_KEY is required");
}

if (!baseUrl) {
  throw new Error("FEATURE_FLAG_API_BASE_URL is required");
}

type CachedFlag = {
  value: unknown;
  refreshedAt: number;
};

const cache = new Map<string, CachedFlag>();

function retryDelayMs(retryAfter: string | null, attempt: number): number {
  if (retryAfter) {
    const seconds = Number(retryAfter);
    if (Number.isFinite(seconds)) return Math.max(0, seconds * 1_000);

    const dateDelay = Date.parse(retryAfter) - Date.now();
    if (Number.isFinite(dateDelay)) return Math.max(0, dateDelay);
  }

  return 100 * 2 ** attempt;
}

function sleep(ms: number): Promise<void> {
  return new Promise((resolve) => setTimeout(resolve, ms));
}

async function fetchRemoteFlag(key: string, timeoutMs: number): Promise<unknown> {
  for (let attempt = 0; attempt < 3; attempt += 1) {
    const controller = new AbortController();
    const timeout = setTimeout(() => controller.abort(), timeoutMs);

    try {
      const response = await fetch(
        `${baseUrl}/v1/flags/get_value/${encodeURIComponent(key)}`,
        {
          method: "GET",
          headers: { Authorization: `Bearer ${apiKey}` },
          signal: controller.signal,
        },
      );

      if (response.status === 429 && attempt < 2) {
        await sleep(retryDelayMs(response.headers.get("retry-after"), attempt));
        continue;
      }

      if (!response.ok) {
        const reason = await response.text();
        throw new Error(`Flag read failed with ${response.status}: ${reason}`);
      }

      return (await response.json()) as unknown;
    } finally {
      clearTimeout(timeout);
    }
  }

  throw new Error("Flag read exhausted its retry budget");
}

async function getFlag(
  key: string,
  localDefault: unknown,
  timeoutMs = 250,
): Promise<{ value: unknown; source: "remote" | "cache" | "local-default" }> {
  try {
    const value = await fetchRemoteFlag(key, timeoutMs);
    cache.set(key, { value, refreshedAt: Date.now() });
    return { value, source: "remote" };
  } catch (error) {
    const cached = cache.get(key);
    if (cached) return { value: cached.value, source: "cache" };

    console.warn("Flag refresh used local default", {
      key,
      reason: error instanceof Error ? error.message : String(error),
    });
    return { value: localDefault, source: "local-default" };
  }
}

const result = await getFlag("checkout_kill_switch", false);
console.log(JSON.stringify(result));
```

Run it with a real environment variable in a TypeScript-capable Node setup. The cache entry includes `refreshedAt` even though the compact return value does not expose it; production telemetry should use that timestamp to report cache age. Add a schema validator once the discovery response schema has been inspected. Do not guess the payload shape.

One subtle failure deserves extra attention. If every invocation starts with an empty cache and immediately performs a remote read, a burst of cold edge instances can become a burst of polling. The code remains correct because it has a local default, but correctness and load are separate questions. Reuse warm-instance state where the runtime permits it, choose a refresh interval that respects acceptable staleness, and put concurrency control around refreshes if several requests can share one process. Your mileage may vary across hosts.

## Monitor the poller, not only the flag value

A flag dashboard showing the desired value cannot prove that clients received it. Monitor the read path itself. A small scheduled probe can call the same flag route with the same timeout policy and emit success, duration, and response status into the team's existing telemetry. Alert there, because the flag API does not provide built-in notifications when reads begin timing out.

Track at least the ratio of refresh outcomes by source and the age of the last-known-good entry. A rising `cache` share is an early warning; a rising `local-default` share says new instances have no successful value to reuse. Break those signals down by runtime and region if the application's own telemetry supports those dimensions. No invented filter parameters are needed on Infrai's logs or metrics APIs, whose discovery parameters do not declare filters. Keep the probe independent from the application request path: if it runs inside the same handler and only when users arrive, low traffic can hide trouble, while a separate schedule supplies evidence even during quiet periods. If the requirement is a heartbeat proving that polling happened on schedule, use a dedicated heartbeat product such as Healthchecks and connect its missed-check signal to the established alert channel. Send the resulting signal to Sentry, Datadog, Grafana, or Better Stack only after checking how the team's current setup represents missed polls, stale cache age, and notification ownership. The flag read and its monitor are two different dependencies, so the acceptance test should stop each one separately.

Clear evidence beats guesswork.

## What are the limits of timeout-safe feature flag polling?

Timeouts and a stale cache improve availability, but they accept bounded staleness. They are not suitable when every evaluation must observe a change immediately, when flag relationships must be enforced by the control plane, or when auditors require a built-in history of each change. A local default also cannot replace evaluation statistics: it tells the application how to behave during uncertainty, not how often users received each variation.

Polling can answer "what value can I safely serve now?" It cannot prove, by itself, "who changed this flag?" or "did every client evaluate it?" Infrai's flag surface lacks change auditing and evaluation statistics, and deleted flags have no recycle bin. For those needs, keep a dedicated system in the shortlist and test it with the same failure scenarios before choosing.

The practical acceptance test is compact. Abort a delayed refresh. Confirm the cached value survives. Start with an empty cache and confirm the deployed default wins. Return `429` with `Retry-After` from a test double and confirm the client waits rather than spins. Finally, stop the probe and confirm the external monitor alerts. Those checks expose the control flow that matters without claiming measured uptime or latency.

## References

- https://logback.qos.ch/manual/appenders.html
- https://healthchecks.io/docs/
- https://docs.launchdarkly.com/sdk/features/config
- https://docs.getunleash.io/reference/sdks
- https://configcat.com/docs/sdk-reference/overview/

## Further reading

- https://developer.mozilla.org/en-US/docs/Web/API/AbortController
- https://nodejs.org/api/globals.html#class-abortcontroller
