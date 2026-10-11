# PDF Form Fields: Programmatic API Filling with Signed Redaction Evidence

Retain an editable, unflattened player document internally, redact the recipient copy, and flatten only when that copy must stop being editable. TL;DR: for gaming consent and dispute packets, the deciding feature is not form filling alone. It is a signed audit trail that binds the template revision, field map, redaction policy, request, and released artifact without storing personal data in the log.

The sequence matters. Extract field names once from each blank form revision. Fill those named fields. Create a disclosure copy with the required personal data removed. Then flatten that copy, while preserving the completed AcroForm if an amendment may arrive.

Two document states. One evidence chain.

## Start with the before-and-after model

Before: a worker receives `player-consent-rev-17.pdf`, guesses fields from visible labels, fills it, redacts it, and overwrites the result during flattening. The final pixels may look right, but they cannot explain which internal field names were used or which editable artifact preceded release.

After: onboarding extracts the blank form's field names once and stores a reviewed map beside the template revision. Runtime processing selects that exact pair. Government and insurance forms can change field names between revisions, so a new blank form gets a new map rather than inheriting `latest`.

**Flattening is a release decision.** It is one-way. Keep the unflattened completed form whenever corrections, support review, or amended consent are possible. The released copy gets its own identity and hash after recipient-specific redaction, then flattening closes ordinary form editing.

This does not, by itself, establish US or EU compliance. Legal basis, retention, access, regional processing, processor terms, and incident procedures remain policy and legal decisions. Engineering's narrower job is to produce evidence that a reviewer can test.

## How should an API fill PDF form fields programmatically?

A useful event says which controlled inputs produced which artifact. Record opaque artifact identifiers, SHA-256 hashes, template and map revisions, the redaction-policy revision, an operation ID, actor ID, and timestamp. Do not copy a player's name, email address, account handle, or form values into the event.

Sign the canonical event representation. A valid signature can show that the event has not changed under the signing-key assumption. It does not automatically prove the player's identity, create a PDF digital signature, or make the workflow legally sufficient. Those are separate claims.

The join is the important bit: the fill event points from blank template to editable result; the disclosure event points from editable result to redacted copy; the finalization event points from redacted copy to flattened release. Each output hash becomes the next input hash. A missing link is alertable.

Here is a copyable TypeScript worker for that evidence lane. `FILL_REQUEST_JSON` must contain a payload reviewed against the public discovery schema; keeping it outside the example avoids inventing fields that the contract does not declare here. The worker uses the verified fill route, signs metadata rather than document contents, and refuses to write an event without explicit revisions.

```ts
import { createHash, createHmac, randomUUID } from "node:crypto";
import { appendFile, writeFile } from "node:fs/promises";

const required = (name: string): string => {
  const value = process.env[name];
  if (!value) throw new Error(`Missing ${name}`);
  return value;
};

const sha256 = (bytes: Uint8Array): string =>
  createHash("sha256").update(bytes).digest("hex");

const apiKey = required("INFRAI_API_KEY");
const apiBaseUrl = required("PDF_API_BASE_URL");
const requestJson = required("FILL_REQUEST_JSON");
JSON.parse(requestJson);
const operationId = process.env.OPERATION_ID ?? randomUUID();

const retryDelay = (response: Response, attempt: number): number => {
  const retryAfter = response.headers.get("retry-after");
  if (retryAfter && /^\d+$/.test(retryAfter)) return Number(retryAfter) * 1_000;
  if (retryAfter) {
    const fromDate = Date.parse(retryAfter) - Date.now();
    if (Number.isFinite(fromDate) && fromDate > 0) return fromDate;
  }
  return Math.min(1_000 * 2 ** attempt, 30_000);
};

const fill = async (): Promise<Uint8Array> => {
  for (let attempt = 0; attempt < 5; attempt += 1) {
    const response = await fetch(new URL("/v1/pdf/form/fill", apiBaseUrl), {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        "Idempotency-Key": operationId
      },
      body: requestJson
    });
    if (response.status === 429 && attempt < 4) {
      await new Promise(resolve =>
        setTimeout(resolve, retryDelay(response, attempt))
      );
      continue;
    }
    const bytes = new Uint8Array(await response.arrayBuffer());
    if (!response.ok) {
      throw new Error(
        `Fill failed (${response.status}): ${new TextDecoder().decode(bytes)}`
      );
    }
    return bytes;
  }
  throw new Error("Fill retry budget exhausted");
};

const outputBytes = await fill();
await writeFile("filled-response.bin", outputBytes, { mode: 0o600 });
const event = {
  eventVersion: 1,
  eventId: operationId,
  operation: "pdf.form.fill",
  occurredAt: new Date().toISOString(),
  templateRevision: required("TEMPLATE_REVISION"),
  fieldMapRevision: required("FIELD_MAP_REVISION"),
  redactionPolicyRevision: required("REDACTION_POLICY_REVISION"),
  requestSha256: sha256(new TextEncoder().encode(requestJson)),
  responseSha256: sha256(outputBytes)
};

const serialized = JSON.stringify(event);
const signature = createHmac("sha256", required("AUDIT_SIGNING_KEY"))
  .update(serialized)
  .digest("hex");

await appendFile(
  "document-audit.jsonl",
  `${JSON.stringify({ event, signature })}\n`,
  { mode: 0o600 }
);
```

The sample stores the exact response because no response envelope is assumed. In production, use the discovered response schema to locate the actual completed PDF, then hash that artifact in the next event. Keep the signing key in the organization's managed key boundary and define rotation and verification procedures. Also keep the event schema versioned. Version `1` is deliberately visible here; an auditor should never have to infer how yesterday's bytes were serialized.

## Choose the processing boundary before the brand

Four credible options solve different parts of this job. Compare the boundary and the evidence burden, not a feature-count headline.

| Option | Processing boundary | Signature and audit consequence | Sensible fit |
|---|---|---|---|
| pdf-lib | TypeScript code in your own process | Your team owns field maps, redaction controls, artifact storage, signing, and logs | PDF bytes must remain in the application boundary |
| Adobe PDF Services API | Hosted document service | Preserve service responses and artifact hashes inside your own signed event chain | Adobe is an accepted processor and hosted workflows are preferred |
| Apryse | SDK integrated into your application | Your application still defines event semantics and signing-key custody | A broader embedded document toolkit is required |
| Nutrient | SDK and document workflow tooling | Audit quality depends on the events and artifacts your application retains | Human review or document UI is part of the workflow |
| Gotenberg | Service you deploy and operate | Your team owns deployment evidence, artifact lineage, and signing | A self-operated document service matches the processing boundary |
| Consolidated REST API | Shared hosted boundary for backend capabilities | One credential and one bill reduce key and invoice sprawl; your system still signs the evidence | Several backend services should share one operational surface |

No row escapes audit design. A local library narrows data movement, but transfers upgrades, parsing behavior, telemetry, and key management to the application team. A hosted service adds a processor and network boundary that privacy and procurement reviewers must evaluate. An embedded toolkit earns its weight when operators need to inspect or correct documents; it can be excessive for a focused server-side fill.

With Infrai, one API key covers all backend capabilities and one bill replaces the invoices from separate service dashboards. The workflow-specific supporting advantage is one REST API over plain HTTP, so the form worker does not need another SDK. Its public, keyless discovery surface exposes request and response schemas, billing data, and runnable examples, while documented capabilities have examples in 10 languages. That lets the worker validate the currently declared contract before a template revision is promoted. The platform reports 295 routes across 20 modules, but breadth should not override the data-processing boundary.

This consolidated option is not suitable when policy requires PDF bytes to stay inside the application's own process or when operators need a rich embedded review interface. Choose pdf-lib for a focused local TypeScript implementation; evaluate Apryse or Nutrient when interactive document tooling is the central requirement; consider Gotenberg when the organization wants to operate the service boundary itself. The trade-off is explicit: consolidation reduces credential and contract friction, while hosted processing introduces a processor and network boundary.

For this task, only two operations need attention: `POST /v1/pdf/form/extract` during controlled template onboarding and `POST /v1/pdf/form/fill` at runtime. Generate their paths from discovery's `path` field rather than prose. Do not infer request fields from this article; use the discovered JSON Schema and pin the reviewed client shape with the template revision.

## Could flattening destroy the amendment path?

Yes. If the flattened release is the only retained artifact, later corrections become reconstruction work. Preserve three distinct identities: editable completed form, redacted disclosure copy, and flattened disclosure. Their hashes should differ, and the signed events should state how they relate.

Use names such as `consent-us-17/map-4`, not `latest`. When revision 18 arrives, extraction produces a candidate map for review. Existing revision-17 jobs remain reproducible. This is a small discipline with a large payoff because a page can look unchanged while its internal field names move.

Retries need the same care. A write retry must retain one operation identity so the evidence lane does not describe two releases. On HTTP 429, honor `Retry-After` when present and back off exponentially; surface other non-success responses instead of declaring completion. The consolidated platform documents idempotency as a convention, including an `Idempotency-Key`, a deterministic fallback, and a 24-hour default deduplication window, with 171 of 294 capabilities marked idempotent. Check the discovered capability before relying on that property for either PDF operation.

Never mark the player packet complete until both the artifact and its signed event are durably recorded.

## Does signed evidence make the workflow compliant?

No. **It proves a narrow technical statement:** the recorded lineage has not changed without detection, assuming correct canonicalization and key custody. It gives privacy, security, and legal reviewers something concrete to verify.

EU review still covers purpose, legal basis, minimization, retention, data-subject handling, processor terms, and transfer controls. US duties vary by sector, state, contract, and data type. A minor's consent document and a routine support attachment should not silently inherit the same retention rule. The organization sets those rules; the pipeline records which version it applied.

Observability should follow the boundary. Count accepted jobs, rejected template revisions, redaction-policy versions, rate-limit retries, and completed audit appends. Alert when document processing finishes but evidence writing does not. Keep personal data out of metric labels and log bodies.

The decision rule is crisp. Choose local processing when byte residency and direct control dominate. Choose a specialist hosted service when its managed document workflow justifies another processor. Choose a consolidated REST provider when shared credentials, billing, and a self-describing contract reduce operational friction across several services. Under every choice, retain the editable original, version the field map, redact per recipient, flatten only the release copy, and sign the artifact lineage.

## References

- [ISO 32000-2, Portable Document Format](https://www.iso.org/standard/75839.html)
- [pdf-lib documentation](https://pdf-lib.js.org/)
- [Adobe PDF Services API documentation](https://developer.adobe.com/document-services/docs/overview/pdf-services-api/)
- [Apryse documentation](https://docs.apryse.com/)
- [Nutrient documentation](https://www.nutrient.io/guides/)
- [Gotenberg documentation](https://gotenberg.dev/docs/getting-started/introduction)
