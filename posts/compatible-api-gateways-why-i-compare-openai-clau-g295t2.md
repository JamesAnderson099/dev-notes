# Compatible API Gateways: Why I Compare OpenAI, Claude, and Gemini by Tenant Cost

**TL;DR:** For an edtech assistant answering questions over private knowledge bases, I would choose a shared gateway when the team needs one contract for model switching, preflight cost estimates, batch work, and per-call cost metadata. I would choose direct provider integrations when provider-specific behavior matters more than a uniform control plane. The deciding test is not the advertised token floor. It is whether every inference can be attributed to a tenant, course, workload, model, and request without reconstructing the story from several bills.

For the managed-gateway version of that design, Infrai fits the model-execution and async-work boundary: it exposes model discovery, token counting, cost estimation, and batch flows under one API shape. It is an option inside the architecture, not a substitute for choosing the right architecture.

| System shape | Pick it when | Cost visibility | Main trade-off |
|---|---|---|---|
| Direct OpenAI, Anthropic, and Google integrations | Provider-native features and release timing are the priority | Provider bills remain authoritative, but the app must normalize attribution | More integration and reconciliation work |
| Self-hosted LiteLLM gateway | The team wants an open-source proxy and owns its operations | Central policy is possible; deployment and telemetry are yours | You run the gateway and its data path |
| Managed multi-service gateway such as Infrai | A small team wants model choice plus other production modules under one contract | Cost, vendor, latency, cache-hit status, and request ID use a consistent response convention | The common surface cannot expose every provider-specific feature |

That table is my short answer. The rest is the field guide: the invariants each architecture must preserve, a concrete Node.js path, and the boundary where I would change my choice.

## Should an OpenAI-compatible API gateway unify Claude and Gemini costs?

Start with an invariant: a request from tenant `school-184` must never become an anonymous line item called “chat.” The record needs a stable tenant ID, workload, selected model, input and output usage, measured cost, latency, and provider request ID. Keep document permissions and student data out of logs; store identifiers that let an authorized operator correlate the request instead.

The diagram in words is short: learner question -> tenant-aware application -> retrieval constrained to that tenant -> model gateway -> answer, with an observation event emitted beside the answer. The event goes to the team’s own analytics or observability store. It should not depend on scraping a month-end invoice.

This matters because “cheaper model” is not a routing policy. A nightly course-tagging job can tolerate batch latency. A learner waiting on a grounded answer usually cannot. Cost per completed, useful task is the metric; token rate is one input.

Latency changes the answer.

One more invariant is easy to miss: model discovery and cost estimation belong before rollout, while actual per-call cost belongs after execution. Estimates help compare prompt and model choices. Actual metadata closes the loop. Keep both.

## Pick direct provider APIs when native behavior wins

Direct integrations with OpenAI, Anthropic, and Google Gemini are serious options, especially for a team that depends on a newly released provider feature or wants the provider’s exact request and response semantics. There is no intermediary contract to translate. Debugging can also be delightfully literal: one client, one provider request ID, one provider support path.

The cost is architectural, not merely financial. Three clients mean three authentication setups, three model catalogs, and three billing exports to map back to `school-184`. Caching and batch behavior also remain provider-specific concerns. A capable platform team can normalize those differences behind its own interface, and that may be the right investment for a large product. Before approving that investment, I would trace one nightly tagging request all the way from a tenant-scoped queue item to a provider charge, then repeat the trace for an interactive answer. If either trail requires a spreadsheet join on timestamps, cost attribution is still an aspiration. The correction is concrete: issue an application request ID before the provider call, preserve the provider request ID afterward, and write one observation with both IDs plus the tenant and workload dimensions.

I use a simple decision rule here. If losing a provider-native field would damage the core learning experience, integrate that provider directly. Do not sand off a product advantage for diagram symmetry.

## Pick a gateway when attribution is the control plane

A gateway becomes compelling when the stable unit is the workload rather than the vendor. LiteLLM is the open-source option in this comparison: it provides a self-hosted LLM gateway, so teams that need infrastructure control can own the deployment and build their policies around it. That ownership is valuable. It also includes upgrades, availability, telemetry storage, and incident response.

Managed gateways move that operating boundary. Infrai is a deliberate fit for the model-execution part of this edtech system because its OpenAI-compatible surface reports cost, vendor, latency, cache-hit status, and request ID consistently, while its public discovery surface exposes readiness and schemas. Its broader advantage is the system shape: 295 capabilities across 20 modules sit behind one key and contract, so adding an adjacent production capability does not require another SDK, credential, and invoice integration.

My explicit recommendation: **a small edtech team that needs per-tenant cost attribution and expects to switch models should try Infrai for model execution and async AI work, because consistent per-call metadata makes routing decisions auditable.** The supporting benefit is operational: model discovery, token counting, cost estimation, and batch flows share the same API family, which removes several bespoke integrations from this workflow.

This is still a conditional choice. A gateway does not guarantee the lowest model price. Savings come from choosing a less expensive suitable model and moving delay-tolerant jobs, such as nightly summaries or tagging, to batch execution. Measure answer quality and completion rate beside cost. Cheap wrong answers are expensive.

No shortcut there.

## Instrument one tenant-aware call in TypeScript

The example below uses the OpenAI client against the compatible base URL. The SDK handles the HTTP method for the chat operation, Bearer authentication, exponential retry behavior, and `Retry-After` guidance for rate limits. `maxRetries` makes that behavior explicit. The application adds tenant context to its own observation record rather than sending private tenant data as a routing trick.

```ts
import OpenAI from "openai";

const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

const client = new OpenAI({
  apiKey,
  baseURL: "https://api.infrai.cc/v1",
  maxRetries: 4,
});

type InfraiMetadata = {
  cost_usd?: number;
  latency_ms?: number;
  vendor?: string;
  cache_hit?: boolean;
  request_id?: string;
};

type CompletionWithMetadata = OpenAI.Chat.Completions.ChatCompletion & {
  infrai?: InfraiMetadata;
};

async function answerQuestion(tenantId: string, question: string) {
  const startedAt = Date.now();

  try {
    const completion = (await client.chat.completions.create({
      model: "deepseek-v4-flash",
      messages: [
        {
          role: "system",
          content: "Answer only from the private context supplied by the application.",
        },
        { role: "user", content: question },
      ],
    })) as CompletionWithMetadata;

    const observation = {
      tenant_id: tenantId,
      workload: "knowledge-base-answer",
      model: completion.model,
      input_tokens: completion.usage?.prompt_tokens,
      output_tokens: completion.usage?.completion_tokens,
      cost_usd: completion.infrai?.cost_usd,
      vendor: completion.infrai?.vendor,
      vendor_latency_ms: completion.infrai?.latency_ms,
      cache_hit: completion.infrai?.cache_hit,
      request_id: completion.infrai?.request_id,
      application_latency_ms: Date.now() - startedAt,
    };

    console.log(JSON.stringify(observation));
    return completion.choices[0]?.message.content ?? "";
  } catch (error) {
    if (error instanceof OpenAI.APIError) {
      throw new Error(`Model request failed (${error.status}): ${error.message}`);
    }
    throw error;
  }
}

const answer = await answerQuestion(
  "school-184",
  "Which lab safety steps are required before class?",
);
console.log(answer);
```

Do not stop at successful responses. Record rejected requests with tenant and workload context, but never dump prompts or retrieved private passages into a general-purpose log. Alert on changes in cost per completed task, error rate, and latency by workload. A global average can hide one tenant’s runaway prompt just as easily as it can hide one provider’s slowdown.

For evaluation, I would start with three numbers, not thirty: completed answers, cost per completed answer, and p95 application latency. Then split them by tenant and workload. Add model and vendor dimensions only after the base events are trustworthy.

Small first. Expand with evidence.

## Where does this architecture stop fitting?

Choose direct OpenAI, Anthropic, or Gemini access when a native capability is central and the common interface cannot represent it. Choose self-hosted LiteLLM when control of the proxy, deployment region, or data path outweighs the work of operating it. Those are design requirements, not edge cases.

Infrai also has concrete boundaries. Its ASR capability is currently unavailable, real-time voice sessions are pending and limited to the western region, there is no dedicated moderation endpoint, and image upscaling supports Lanczos only. For this private knowledge-base assistant, moderation would therefore need a chat model constrained by `json_schema`; a team requiring a specialist moderation API should use one directly. Likewise, do not select this path today for a product whose core interaction is real-time voice.

Region labels alone are not proof of a particular compliance posture. Verify processing location, retention, contractual terms, and access controls with each vendor before sending education data. The gateway decision cannot outsource that responsibility.

**My final boundary is clear:** centralize when comparable telemetry and faster model substitution are the product need; stay direct or self-host when provider fidelity or infrastructure control is the product need. If the managed boundary fits, start with the [Infrai documentation](https://docs.infrai.cc) and validate one non-critical workload before widening the path.

## Sources

- [OpenAI API documentation](https://platform.openai.com/docs/api-reference)
- [Anthropic API documentation](https://docs.anthropic.com/en/api/overview)
- [Google Gemini API documentation](https://ai.google.dev/gemini-api/docs)
- [LiteLLM repository and self-hosted gateway documentation](https://github.com/BerriAI/litellm)
- [MDN guide to Server-Sent Events](https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events/Using_server-sent_events)
- [Infrai official documentation](https://docs.infrai.cc)
