# Stable Feature-Flag Rollouts: A Practical Node.js Express API

**Use deterministic user bucketing in one Express middleware, then observe outcomes by flag variant before increasing the rollout percentage.**

That is the smallest feature-flags API I trust in a Node.js service. The before picture is scattered `if` statements and a percentage that behaves like a fresh coin toss. The after picture is one evaluator: request identity goes in; a boolean and a reason come out; logs and metrics record the decision. Crisp. Testable. Reversible.

This note builds that path without tying the application to a flag vendor. It also draws a hard boundary around what the simple design cannot do. Feature toggles create alternate code paths, and Martin Fowler's feature-toggle overview distinguishes release, experiment, ops, and permissioning uses. Those jobs have different lifetimes and controls. Treating all of them as one boolean map is where the easy implementation starts lying to its operators.

## What should the simple rollout mental model be?

Picture five boxes in a line: authenticated request, targeting rules, stable bucket, application branch, telemetry. Identity enters once. The evaluator first checks an explicit targeting rule, then places everyone else into one of 100 buckets. The route consumes the decision but never reimplements it. Finally, telemetry records the flag key, variant, and evaluation reason.

The important word is stable. A random number per request can put the same person in the new checkout on Monday morning and the old checkout that afternoon. Instead, hash the flag key with a durable user identifier and map the result to `0` through `99`. A 10% rollout enables buckets below 10. Moving to 25% preserves the first group and adds buckets 10 through 24. Including the flag key prevents every experiment from selecting the same users.

Targeting comes before bucketing in this example. That lets an internal account or a named cohort receive the feature while the general rollout remains at 0%. The returned reason matters just as much as the boolean — when support asks why one account saw a branch, `target:plan` is an answer and `true` is not.

| Decision point | Simple local evaluator | Control-plane-backed evaluator |
| --- | --- | --- |
| Configuration change | File or internal config delivery | Managed change workflow |
| Stable percentage | Deterministic hash | Must verify the provider's semantics |
| User targeting | Rules defined in code or config | Rules managed outside the deploy |
| Audit and approvals | Build them yourself | Choose this route when they are required |
| Runtime dependency | Local evaluation | Depends on the chosen architecture |

This is a method comparison, not a product ranking. The local version wins on inspectability. Its catch is governance.

## How can a Node.js feature flags API implement simple rollout percentage and user targeting in Express?

Keep evaluation pure, validate the rollout range, and attach decisions once per request. Here is the copy-pasteable core. All identities come from an authentication layer in this example; don't accept an untrusted client header as an account identity.

```ts
import { createHash } from "node:crypto";
import express, { type NextFunction, type Request, type Response } from "express";

type Context = {
  userId: string;
  plan?: string;
};

type TargetRule = {
  attribute: "plan";
  values: string[];
};

type Flag = {
  key: string;
  enabled: boolean;
  rolloutPercentage: number;
  targets: TargetRule[];
};

type Decision = {
  enabled: boolean;
  reason: "flag_off" | "target:plan" | "percentage";
  bucket?: number;
};

function bucket(flagKey: string, userId: string): number {
  const digest = createHash("sha256")
    .update(`${flagKey}:${userId}`)
    .digest();

  return digest.readUInt32BE(0) % 100;
}

function evaluate(flag: Flag, context: Context): Decision {
  if (!flag.enabled) return { enabled: false, reason: "flag_off" };

  const targeted = flag.targets.some((rule) => {
    const value = context[rule.attribute];
    return value !== undefined && rule.values.includes(value);
  });

  if (targeted) return { enabled: true, reason: "target:plan" };

  const userBucket = bucket(flag.key, context.userId);
  return {
    enabled: userBucket < flag.rolloutPercentage,
    reason: "percentage",
    bucket: userBucket,
  };
}

const flags: Flag[] = [
  {
    key: "new-checkout",
    enabled: true,
    rolloutPercentage: 10,
    targets: [{ attribute: "plan", values: ["internal"] }],
  },
];

const app = express();

app.use((req: Request, res: Response, next: NextFunction) => {
  const user = res.locals.authenticatedUser as Context | undefined;
  if (!user) return res.status(401).json({ error: "authentication_required" });

  const decisions = new Map(
    flags.map((flag) => [flag.key, evaluate(flag, user)]),
  );

  res.locals.flagDecisions = decisions;
  next();
});

app.get("/checkout", (_req: Request, res: Response) => {
  const decision = res.locals.flagDecisions.get("new-checkout") as Decision;
  res.json({ checkout: decision.enabled ? "new" : "current" });
});
```

Validate configuration before replacing the in-memory snapshot: percentage must be an integer from 0 through 100, keys must be unique, and every rule attribute must be allowed. If a refresh is invalid, keep the last validated snapshot and alert on the rejected update. The request path should only read a complete snapshot; mutating individual rules while requests are evaluating them makes explanations unreliable.

Write unit tests around boundaries, not distribution luck. Pin a few known `(flagKey, userId)` pairs to expected buckets, assert that a disabled flag overrides targeting, assert that targeting overrides 0%, and assert that bucket 9 is on at 10% while bucket 10 is off. That gives future implementations a compatibility contract.

## Observe the decision and the outcome

An exposure count proves that evaluation happened. It does not prove that the rollout helped. I teach this as two joined records: the decision says who received the branch and why; the outcome says what happened afterward. Join them through a request or trace identifier in logs, and aggregate operational metrics by flag key and variant. Keep user IDs out of metric labels because distinct identities create an unbounded label set.

For server behavior, compare request count, error count, and latency between `current` and `new`. For a browser-facing change, compare the relevant user-experience measurements by variant. The web.dev Core Web Vitals guidance defines LCP, INP, and CLS and recommends assessing the 75th percentile separately for mobile and desktop. A server flag can still change shipped JavaScript or rendering behavior, so backend health alone can miss the regression.

I've paid for the noisier lesson. During one rollout, I estimated about $160 for the month, but the bill reached $1,840 because 12 flag checks each emitted a full JSON exposure event on every request. Nothing in the evaluator was wrong. My telemetry design was. I replaced per-check logs with low-cardinality counters, retained sampled decision logs for investigation, and made the sampling stable by user so a selected session stayed inspectable from start to finish.

Don't log everything.

Alerting should compare outcomes, not the raw percentage. A useful rollout gate might require enough observations to avoid reacting to one request, then stop an increase when the new branch's error ratio or latency crosses a team-owned threshold. I can't give a universal threshold because traffic shape, risk, and baseline noise differ; your mileage may vary. Write the threshold down before the rollout anyway. Otherwise people reinterpret the graph after seeing the result.

The flag key also needs a bounded lifecycle. Put an owner and removal date beside the definition, search for both branches during cleanup, and test the off path until the flag is deleted. Alternate paths are inventory. Inventory ages.

## When is this design not suitable, and what should replace it?

The first objection I hear is, "Why not use a database and make changes instantly?" You can, but instant mutation introduces distribution questions: cache duration, partial refreshes, stale processes, and what happens when configuration cannot be read. The local snapshot is suitable when engineers own a small set of release flags and the existing deployment or configuration channel is fast enough. It is not suitable when a kill switch must propagate under a strict deadline across many services. In that case, choose a dedicated control plane or build a configuration service with versioned snapshots, authentication, audit events, and an explicit stale-data policy.

The second objection is, "Can this run experiments?" It can assign a stable cohort, but assignment is only one part of experimentation. Exposure timing, sample-ratio checks, outcome definitions, exclusion rules, and statistical analysis still need ownership. Stick with a purpose-built experimentation workflow when the decision is about causal impact rather than a guarded release. A homegrown boolean and a dashboard don't settle that question.

There are more boundaries. This exact TypeScript evaluator won't guarantee cross-language consistency unless every service uses the same byte encoding, digest slice, modulo operation, flag key, and canonical identity. Centralize evaluation or publish test vectors before another language joins. Permission flags also deserve a separate authorization system; a release toggle should not become the only control protecting sensitive data. And if non-engineers must schedule changes, approvals are mandatory, or auditors need a durable history, config in source control may be too blunt.

My practical rule is short: own this implementation while the problem is deterministic release control; replace it when the real problem becomes governance, rapid propagation, authorization, or experimentation. Keep the application call site narrow so that replacement doesn't rewrite every route. The evaluator may change. The contract — context in, typed decision and reason out — can stay.

## References

- Martin Fowler, "Feature Toggles (aka Feature Flags)": https://martinfowler.com/articles/feature-toggles.html
- web.dev, "Web Vitals": https://web.dev/articles/vitals
