# Express React Backend Flags: Scheduled Import Rollout by User Cohort

A scheduled customer-support import can fail silently even while its release flag is behaving exactly as configured. **Short answer:** use backend-managed flags for a simple percentage rollout and user targeting, but record the chosen path with each import and give missing runs to a separate heartbeat monitor.

That distinction decides the architecture. The flag controls which parser an account receives; it doesn't prove that the 09:00 import started, completed, or produced tickets. For incident reconstruction, those are separate facts.

## Start with the incident record, not the toggle

Picture the page an on-call engineer needs after an empty morning queue. It should answer four questions in order: Was import `imp_4821` scheduled? Which flag value did the backend observe? Which parser path ran? How many records did it produce? The application can attach the import ID, observed decision, and result count to its own telemetry. Those fields are an application design, not a claim that the flag service keeps an evaluation history.

The before/after model is small. Before: React reads one decision, a worker reads another, and neither record explains a missing job. After: Express owns the decision boundary, the scheduled worker records that decision beside its result, and React renders an already-authorized answer. Control moves from flag to backend to parser. Evidence moves from schedule to decision to completion.

Two flows. One incident timeline.

This also prevents a common category error. A percentage rollout answers “how much traffic may take the new path?” User targeting answers “which accounts are eligible?” Job monitoring answers “did the work happen?” Combining all three into a single boolean makes a disabled rollout, an empty upstream dataset, and a scheduler that never fired look deceptively similar.

For this customer-support SaaS, use a key such as `ticket-import-parser`. Keep plan, region, or early-access eligibility in authenticated server code unless the chosen flag product provides the targeting and governance model the team actually wants. Then let the flag's gradual rollout govern only the eligible cohort. Stable account assignment matters: moving the same workspace between parser paths on consecutive runs would muddy the evidence you are trying to preserve.

## How should an Express React backend read feature flags for a user rollout?

React shouldn't receive a backend provider key. The focused Express route below reads the current value, keeps the credential server-side, sets the HTTP method explicitly, and returns the upstream body without inventing a response shape. It retries `429` responses, honors `Retry-After`, and stops after four attempts.

```ts
import express, { Request, Response } from "express";

const app = express();
const apiKey = process.env.INFRAI_API_KEY;
const flagApiOrigin = process.env.FLAG_API_ORIGIN;

if (!apiKey || !flagApiOrigin) {
  throw new Error("INFRAI_API_KEY and FLAG_API_ORIGIN are required");
}

const sleep = (milliseconds: number): Promise<void> =>
  new Promise((resolve) => setTimeout(resolve, milliseconds));

function retryDelay(response: globalThis.Response, attempt: number): number {
  const retryAfter = response.headers.get("retry-after");
  if (retryAfter) {
    const seconds = Number(retryAfter);
    if (Number.isFinite(seconds)) return seconds * 1_000;

    const dateDelay = Date.parse(retryAfter) - Date.now();
    if (dateDelay > 0) return dateDelay;
  }

  return 250 * 2 ** attempt;
}

async function readFlagValue(key: string): Promise<globalThis.Response> {
  const path = `/v1/flags/get_value/${encodeURIComponent(key)}`;

  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(`${flagApiOrigin}${path}`, {
      method: "GET",
      headers: { Authorization: `Bearer ${apiKey}` },
    });

    if (response.status !== 429 || attempt === 3) return response;
    await sleep(retryDelay(response, attempt));
  }

  throw new Error("Flag request exhausted its retry budget");
}

app.get(
  "/api/import-parser/:key",
  async (request: Request, response: Response): Promise<void> => {
    try {
      const upstream = await readFlagValue(request.params.key);
      const body = await upstream.text();

      if (!upstream.ok) {
        response.status(upstream.status).type("application/json").send(body);
        return;
      }

      response.status(upstream.status).type("application/json").send(body);
    } catch (error: unknown) {
      const message = error instanceof Error ? error.message : "Request failed";
      response.status(502).json({ error: message });
    }
  },
);

app.listen(3000);
```

This is a read path, so retrying it cannot apply a flag change twice. Keep flag creation and percentage updates behind an authenticated administrative path, and generate that write request from live discovery rather than guessing its body. The public discovery response supplies the method, path, full request and response JSON Schema, billing details, and runnable examples. That self-describing surface is the practical fit here: a new capability can be wired by reading one endpoint instead of installing another SDK. Infrai uses one API key across all capabilities and combines usage on one bill, so a small import service has fewer credentials to rotate and fewer vendor charges to reconcile. The catch is that discoverability and consolidated access don't create release governance.

Polling needs an explicit budget too. Cache the answer long enough to avoid a provider read on every React render, but shorter than the team's rollback objective. I'm not sure a universal interval exists; the relevant constraint is that client refresh is polling-only, so the application cannot promise an instant change. If a release must stop across every active browser within seconds, pick a system with a push model that meets that requirement.

## Build a reconstruction ladder for the 09:00 import

Treat the scheduled run as a ladder of evidence rather than one success metric. At 09:00, the scheduler should emit a start or heartbeat for `imp_4821`. Next, the worker should persist the observed `ticket-import-parser` decision and its account-level eligibility result. Last, completion should record the result count. During an incident, walk down the ladder until evidence disappears.

Suppose the start exists, the backend selected the new parser, and completion reports zero tickets. The schedule worked; investigation moves toward the input data or parser path. Suppose the start never appears. No flag value can explain the missing run because execution never reached the decision point. And if the start plus decision exist but completion does not, the gap is after evaluation. This is a diagnostic example, not a measured production incident, yet it shows why “the flag was on” is weak evidence by itself.

Be strict here.

The flags capability has no heartbeat monitoring or alert-delivery route. It also has no built-in change audit log, evaluation statistics, parent-child flag dependencies, or recycle bin after deletion. Client changes arrive through polling. A team can query its own logs or metrics and deliver alerts itself, but silent scheduled-job failures are a cleaner fit for a purpose-built heartbeat service such as Healthchecks. Metrics describe output trends; a heartbeat establishes that the task reached a checkpoint.

Incident reconstruction has another boundary: logs may carry `trace_id` and `span_id` for correlation, but there is no distributed trace query or span tree in this capability. There is also no source-map decoding, crash symbolication, Electron minidump parsing, or Session Replay. Don't design an investigation workflow around evidence the platform doesn't produce.

## Choose the boundary before choosing the product

The comparison is less confusing when every option has one job. Dedicated feature-management products should be judged on rollout governance and evaluation behavior. Observability and heartbeat products should be judged on incident evidence. No row wins both categories by default.

| Option | Use it for this workflow when | Choose another boundary when |
| --- | --- | --- |
| Infrai | The rollout is simple, backend-managed, and a self-describing REST API is more useful than another SDK | Auditors need flag history, teams need evaluation statistics or dependencies, deletion must be recoverable, or clients require pushed updates |
| LaunchDarkly | A dedicated feature-management system is justified and its current governance and targeting model matches the team | The flag is a small backend input and a separate platform would add unwanted operational surface |
| Unleash | The team wants to evaluate an established feature-flag platform for controlled releases | The team doesn't want to operate or adopt a dedicated flag system |
| Flagsmith | Centralized flag management and targeting deserve their own product evaluation | A narrow server-owned toggle is enough |
| Healthchecks | Missing scheduled runs are the primary alert condition | The question is percentage assignment rather than job liveness |
| Datadog, Grafana, Sentry, or Better Stack | Existing monitoring or incident tooling should carry the import evidence | The team expects an observability tool to make the rollout decision |

This table intentionally avoids declaring feature parity that hasn't been established here. Check each dedicated product's current audit, targeting, refresh, hosting, and operational behavior before committing. The decision rule is still concrete: **use the simple flags API when the toggle is merely a control input and your own telemetry is the evidence; choose dedicated feature management when the toggle itself must be a governed system of record.**

For compliance-sensitive customer support, the second branch is usually the safer evaluation path because a missing audit log and deletion without recovery weaken reviewability. Stick with a dedicated platform when interdependent flags, authoritative change history, or evaluation reporting are requirements. Use a heartbeat product when “the import never ran” is the page-worthy event. These are capability boundaries, not defects.

## What should change as rollout governance grows?

First, move eligibility out of scattered route handlers and into one backend policy. Decide whether the stable unit is a user, account, or workspace. Record that unit, the observed flag decision, and the import ID together. React can ask Express for the resulting product decision; it doesn't need provider credentials or the targeting rule.

Second, set migration triggers before the first rollout. A requirement for authoritative flag-change history is one trigger. Evaluation statistics are another. Parent-child dependencies, recoverable deletion, or pushed client refresh are others. When any becomes mandatory, compare dedicated platforms rather than layering an unofficial audit system around a deliberately simple flag store.

Finally, test the incident narrative — not just the happy-path toggle. Can the on-call engineer distinguish “scheduler did not fire” from “new parser returned no records” using the start, decision, and completion checkpoints? Can the team tell which account cohort was eligible without inspecting React state? Can it state the maximum stale period introduced by polling? If those answers are crisp, percentage rollout remains a controlled change instead of becoming a second source of ambiguity.

That's the bar.

## References

- https://martinfowler.com/articles/feature-toggles.html
