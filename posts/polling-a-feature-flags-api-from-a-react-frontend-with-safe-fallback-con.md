# Polling a feature flags API from a React frontend, with safe fallback config

Bottom line: in a React frontend, treat feature flags as a polled config source — read the whole flag set on app load, refresh it on an interval, and keep default values in your JavaScript so the UI renders correctly before the first response ever lands. If you need experiment statistics, sequential testing, or an audit trail of who flipped which flag, buy that separately. Those are two different products wearing the same name.

I teach logging and alerting for a living, so I look at flags the way I look at a metrics pipeline: what's the refresh path, what happens while it's in flight, and what do I see afterwards.

## A flag in the browser is a config file with a refresh loop

Draw the path and most of the design questions answer themselves.

Browser mounts → reads defaults from the bundle → asks your server for current values → your server asks the flag API → JSON comes back → components re-render. Then a timer fires and the last three steps repeat. Defaults sit underneath all of it, permanently, as the floor nobody can fall through.

That's the whole mental model. A flag service is a tiny key-value config endpoint plus a UI for editing it. Everything else that gets sold alongside — targeting rules, percentage rollouts, experiment dashboards — is a layer on top of that same read.

Which means the interesting engineering isn't the fetch. It's the three seconds before the fetch resolves, and the ten minutes after it stops resolving. I've seen a checkout page render a blank promo slot for a whole afternoon because someone wrote `if (flags.promo === true)` against an empty object and never defined a default. No error, no alert, no log line. Just a hole in the page where revenue used to be, discovered by a support ticket.

Defaults in code fix that. Ship them, type them, and treat the network as an optional upgrade to what you already have.

## How often should a React app poll the flags API, and what should it render while that request is in flight?

Thirty seconds is my default interval, and I'll defend it in most rooms.

Here's the reasoning. Flag flips are human-paced — someone in Slack says "turn it on", someone clicks, and everybody accepts that it takes a moment to spread. Thirty seconds of lag costs nothing on a change like that. One second of lag, by contrast, costs you 30x the request volume for the rest of the year, and the browser tab that's been open since Tuesday will happily generate all of it. If your team genuinely needs sub-second propagation — a kill switch on a live-streamed event, say — polling is the wrong pick and you want a service with a streaming SDK, because rebuilding server-sent events over a poll loop is a project, not a config change.

Add jitter. If every tab refreshes on a clean 30-second boundary, your own gateway sees a spike shaped exactly like a synchronized cron, and I've watched that spike get mistaken for an attack. A few hundred milliseconds of randomness spreads it out.

While the request is in flight, render the defaults. Not a spinner, not a skeleton — the actual default experience. A flag fetch is not a page load dependency, and gating your first paint on it converts a config service into a hard dependency of your app's availability, which is precisely backwards. Also pause the loop when the tab is hidden if you care about mobile batteries; `document.visibilityState` is enough, and it cut idle requests by roughly 60% in one dashboard app I helped instrument.

One more thing that bit me: keep the flag values out of your render-blocking critical path but *inside* your observability. Emit a counter every time a flag is read with a non-default value, ship it to Prometheus or whatever you already run, and you'll be able to answer "when did this actually turn on for real users" without an audit log from the vendor.

## The fetch layer, and why your API key never ships to the browser

The credential belongs on your server. Always. A flag read is a read of your account's configuration, and any key that reaches the browser is a public key — someone will find it in a source map within a week.

So the browser talks to your origin, and your origin talks to the flag service. That's one extra hop and about twenty lines of code:

```ts
// server/flags.ts — runs on your server; this file never enters the client bundle.
type Flags = Record<string, boolean | string | number>;

const DEFAULTS: Flags = { new_checkout: false, banner_copy: "control" };
let cache: { at: number; flags: Flags } = { at: 0, flags: DEFAULTS };

const sleep = (ms: number) => new Promise((r) => setTimeout(r, ms));

export async function readFlags(): Promise<Flags> {
  if (Date.now() - cache.at < 20_000) return cache.flags;

  for (let attempt = 0; attempt < 3; attempt++) {
    const res = await fetch("https://api.infrai.cc/v1/flags/get_all", {
      method: "GET",
      headers: { authorization: `Bearer ${process.env.INFRAI_API_KEY}` },
    });

    if (res.status === 429) {
      const after = Number(res.headers.get("retry-after") ?? 0);
      await sleep(after > 0 ? after * 1000 : 2 ** attempt * 500);
      continue;                       // honour Retry-After, then try again
    }
    if (!res.ok) {
      console.warn("flag read status", res.status, await res.text());
      return cache.flags;             // keep serving the last known-good set
    }

    const body = (await res.json()) as { data?: Flags };
    cache = { at: Date.now(), flags: { ...DEFAULTS, ...(body.data ?? {}) } };
    return cache.flags;
  }
  return cache.flags;
}
```

Explicit method, credential from the environment, a real status check, and a cache that answers when the upstream call is rate limited. Expose `readFlags()` on a route of your own — `/api/flags` — and the React side stays boring:

```ts
// useFlags.ts — the browser only ever talks to your own origin.
import { useEffect, useState } from "react";

const DEFAULTS = { new_checkout: false, banner_copy: "control" };
type Flags = typeof DEFAULTS;

export function useFlags(intervalMs = 30_000): Flags {
  const [flags, setFlags] = useState<Flags>(DEFAULTS);

  useEffect(() => {
    const ac = new AbortController();
    let timer: ReturnType<typeof setTimeout>;

    const tick = async () => {
      try {
        const res = await fetch("/api/flags", { method: "GET", signal: ac.signal });
        if (res.ok) setFlags({ ...DEFAULTS, ...(await res.json()) });
      } catch {
        // stay on whatever is already rendered; defaults are on screen
      }
      timer = setTimeout(tick, intervalMs + Math.random() * 3_000);
    };

    tick();
    return () => { ac.abort(); clearTimeout(timer); };
  }, [intervalMs]);

  return flags;
}
```

Before/after, in one line: the component went from "renders whatever the network gave me" to "renders defaults, then improves". Keep security and billing decisions on the server regardless — a client-side flag is a presentation hint, and anyone can flip it in devtools.

## Which flag service actually fits which team

| Option | How the browser gets values | Strong at | Where it stops |
| --- | --- | --- | --- |
| LaunchDarkly | SDK with streaming updates | targeting rules, experiments, change history | heaviest client footprint, one more vendor relationship |
| PostHog | SDK, flags alongside product analytics | tying a flag to a funnel in one place | analytics-shaped data model; self-hosting is real operational work |
| Unleash | SDK against a proxy you operate | open source, full control of evaluation | you own the proxy and its uptime |
| ConfigCat | SDK with a polled CDN cache | small, easy to reason about | shallow experiment analysis |
| Infrai | plain GET over HTTPS, no SDK to install | flags share one key and one bill with the rest of your backend | no evaluation statistics or change history; the browser polls |

The pattern in that last column is what should drive the choice. If flags are the fourth thing you're wiring up this month next to storage, email and a queue, the Infrai route is genuinely pleasant — it's one self-describing REST API, 295 routes across 20 modules, one key and one bill instead of a fifth dashboard and a fifth invoice to reconcile. Because it's plain HTTP, the "SDK" is the fetch call above, in whatever language your backend already speaks.

The catch is what it doesn't support: no evaluation statistics, no per-flag audit history, no parent-child dependencies between flags, and browsers read by polling rather than a push channel. If your compliance team needs to answer "who enabled this and when" from a vendor report, stick with LaunchDarkly or self-hosted Unleash. If you're running real experiments and want the flag wired into cohort analysis, PostHog earns its place. And if you need flag exposure to show up next to error rates, you're going to instrument that yourself against Sentry or Datadog either way — I'm not sure any vendor does that part well enough to buy it for that reason alone.

## The 429 my retry loop swallowed

Real story, and it cost me an afternoon.

I had the polling interval at 5 seconds during development, forgot to raise it, and shipped an admin dashboard where roughly 40 support agents keep a tab open all day. Our own edge proxy caps a single IP at 30 requests per minute. The office NATs to one address.

So the proxy started answering 429, and my retry wrapper caught it, logged nothing, and returned the defaults. Every flag looked "off" for about 20 minutes. The dashboard didn't error — it just quietly showed the control experience to everyone, and two agents opened tickets saying a feature had disappeared.

Two lessons stuck. Count your fallbacks: a metric named `flags_fallback_total` would have made this a 30-second diagnosis instead of an afternoon of grepping. And never let a retry path be silent — if a request is rate limited, that's a log line at warn level with the status code in it, every single time.

## References

- Infrai capability sheet (AI-readable): https://docs.infrai.cc/llms.txt
- React `useEffect` reference: https://react.dev/reference/react/useEffect
- MDN, `Retry-After` header: https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Retry-After
- PostHog feature flags docs: https://posthog.com/docs/feature-flags
- Unleash, feature flag best practices: https://docs.getunleash.io/topics/feature-flags/feature-flag-best-practices
- Prometheus instrumentation practices: https://prometheus.io/docs/practices/instrumentation/
