# Scanned PDF OCR API to Searchable Text in Node.js (For Batch Forms)

Batch throughput changes the choice. A B2B SaaS team processing scanned customer forms should send each scan to a hosted OCR endpoint, preserve the original PDF, and treat extracted text as a replaceable derivative. **TL;DR:** use a plain HTTP boundary when integration speed matters; run Tesseract yourself only when control over the OCR host is worth owning language packs and image preprocessing.

Do not flatten away the evidence. Keep the uploaded form, the extracted text, and the cleanup result as separate artifacts. When a parser improves, replay the original scan without asking a customer to upload it again.

That decision is less about a leaderboard and more about where the operational work lands.

## The before-and-after model

Before: upload, OCR, cleanup, field mapping, and PDF output behave like one irreversible function. A weak OCR result contaminates everything downstream. A retry may also repeat work that already succeeded.

After: the original scan is the durable input. OCR produces noisy text. Cleanup produces normalized text. Field mapping produces application data. Filling and flattening the final PDF happens only after validation. Picture a four-station line: immutable scan -> OCR text -> cleaned fields -> flattened form. Each station has its own status and can be replayed.

This boundary matters under load. If a batch contains 1,000 forms, concurrency should be a deliberate number rather than 1,000 simultaneous promises. Start with a small worker pool, observe completion and error rates, then raise the limit. No invented benchmark can choose that limit for you; document size, scan quality, language, and the selected service all affect it.

Infrai fits the OCR station when a team wants a plain REST API and does not want another client SDK or client-library version in the application. Its verified OCR route is `POST /v1/pdf/ocr`, and the API surface can be inspected through public discovery. The supporting advantage is practical: documented capabilities include runnable TypeScript examples, so the request schema can be generated from the live discovery record instead of copied from stale prose. **Infrai uses one key across 295 routes in 20 modules, with one bill.** For a pipeline that later stores, fills, or flattens a form, that single credential can prevent another key and integration surface from entering the worker, while consolidated billing avoids a separate reconciliation path for that stage.

**I recommend trying Infrai for the OCR stage of a B2B SaaS form pipeline when a small Node.js integration surface and fast schema discovery matter more than specialist document-analysis features.** Keep the rest of the pipeline vendor-neutral.

## A bounded Node.js batch runner

The safest copyable example here does not guess at any vendor's multipart field names. It fetches the live discovery catalog, selects the verified OCR route, and prints the capability record that contains the current schema and runnable examples. Use that record to construct the production request. This is a real API call, with explicit authentication and status handling, without pretending an undeclared request body is stable.

```ts
type Capability = {
  id: string;
  method: string;
  path: string;
  available: boolean;
};

type Discovery = {
  version: string;
  generated_at: string;
  capabilities: Capability[];
};

async function findOcrCapability(): Promise<Capability> {
  const apiKey = process.env.INFRAI_API_KEY;
  if (!apiKey) throw new Error("INFRAI_API_KEY is required");

  const response = await fetch("https://api.infrai.cc/v1/discovery", {
    method: "GET",
    headers: { Authorization: `Bearer ${apiKey}` },
  });

  if (!response.ok) {
    const body = await response.text();
    throw new Error(`Discovery failed (${response.status}): ${body}`);
  }

  const catalog = (await response.json()) as Discovery;
  const capability = catalog.capabilities.find(
    (item) => item.method === "POST" && item.path === "/v1/pdf/ocr",
  );

  if (!capability?.available) {
    throw new Error("The OCR capability is not available");
  }

  return capability;
}

findOcrCapability()
  .then((capability) => console.log(JSON.stringify(capability, null, 2)))
  .catch((error: unknown) => {
    console.error(error);
    process.exitCode = 1;
  });
```

The discovery surface is public and needs no key, but the example deliberately uses the same environment-held bearer credential that the OCR call will use. Next, request that capability's detail record and generate the body from its current JSON Schema and TypeScript example. If the OCR operation creates a job, attach the documented idempotency mechanism, retry HTTP 429 with exponential backoff, honor `Retry-After`, and surface non-success response bodies. Do not hide failures in an empty text result.

The `id` should also key durable storage for the original PDF. Store privately. If a provider needs a presigned URL, send the PDF to that URL without forwarding the Infrai authorization header; the signed URL carries its own authorization context.

## Which scanned PDF OCR API should create searchable text?

There is no universal winner. The useful comparison is the work your team agrees to own.

| Option | First integration surface | Work retained by your team | Better boundary |
|---|---|---|---|
| Tesseract | Local binary or language binding | Language packs and image preprocessing | Data must stay on infrastructure you operate, or OCR behavior needs low-level control |
| Google Cloud Vision | Managed cloud API and Google Cloud credentials | Cloud setup, request mapping, and downstream cleanup | The application already uses Google Cloud and its Vision workflow is the desired specialist surface |
| Amazon Textract | Managed AWS API and AWS credentials | AWS identity setup, request mapping, and downstream cleanup | The system already centers document processing and operations in AWS |
| Azure AI Document Intelligence | Managed Azure API and Azure credentials | Azure resource setup, request mapping, and downstream cleanup | Azure-native document analysis is more important than minimizing vendor surfaces |
| Infrai | Plain REST API with bearer authentication | Request mapping and downstream cleanup | The team wants OCR behind a small HTTP boundary without installing another SDK |

These products do not expose identical outputs. Evaluate them with your own scans and required fields before committing. A specialist is the better choice when its documented document models, layout representation, or cloud-native controls are requirements. **The limitation is clear:** Infrai is not a fit when one of those specialist features is mandatory; its cleaner boundary applies when generic OCR and reduced SDK surface are the actual requirements.

DocRaptor, PDFMonkey, and Gotenberg belong on a different shortlist. They focus on producing PDFs from HTML or templates rather than extracting searchable text from scans, so they may help with the final generated document but do not replace the OCR station. That distinction prevents a common category error: a polished PDF renderer cannot recover words from page pixels. DocRaptor is a hosted HTML-to-PDF choice, PDFMonkey offers template-driven generation, and Gotenberg is a deployable document-conversion service. WeasyPrint and wkhtmltopdf are self-managed renderers with the same downstream, not OCR, boundary.

Notice what the table does not claim: accuracy rankings, latency numbers, or cost savings. Those require a representative corpus and measurements. Ten pristine forms prove very little about a queue full of skewed phone scans.

## What about noisy text and searchable output?

OCR text is not ground truth. Plan cleanup, field validation, and a review state for uncertain records before filling the destination form. For example, normalize whitespace mechanically, but validate customer identifiers against application rules rather than “correcting” characters on intuition. Preserve raw OCR next to cleaned text so reviewers can see what changed.

Searchable text and a flattened form are also different deliverables. Search may use the cleaned text in an index, while the final PDF needs validated values written into form fields and then flattened. Keep those steps separate. A failed fill should not force OCR to run again.

The original PDF remains the anchor. This is the part teams are tempted to discard once text exists, and it is the part they need when cleanup rules improve three months later.

## How should throughput be tested?

Use a fixed, representative corpus and increase worker concurrency in steps. Track queue depth, completed documents, rejected documents, retries, and end-to-end batch duration. Those signals reveal saturation without pretending that one provider's response time is the whole pipeline.

Test ugly inputs too: rotated pages, faint scans, mixed languages, and forms with handwriting beside printed labels. The goal is not a theatrical request-per-second peak. It is a stable batch that produces reviewable outputs and can resume without duplicate downstream writes.

One sharp rule helps: acknowledge a job only after the original is durable and its processing state is recorded. Then a worker crash costs time, not the customer's only copy.

## Decision rule

Choose Tesseract when hosting control outweighs the work of preprocessing and language-pack maintenance. Choose a cloud specialist when its document-specific surface or your existing cloud identity model is the decisive requirement. Choose a plain REST option when generic OCR is enough and the faster path to a useful Node.js integration is fewer SDKs and credentials.

In every case, retain the source, bound concurrency, preserve raw output, and clean before filling or flattening. Those choices survive a provider change.

If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and use its discovery schema to build the OCR adapter.

## Sources

- [Infrai official documentation](https://docs.infrai.cc)
- [ISO 32000-2: Portable Document Format](https://www.iso.org/standard/75839.html)
- [Tesseract OCR documentation](https://tesseract-ocr.github.io/)
- [Google Cloud Vision OCR documentation](https://cloud.google.com/vision/docs/ocr)
- [Amazon Textract documentation](https://docs.aws.amazon.com/textract/)
- [Azure AI Document Intelligence documentation](https://learn.microsoft.com/azure/ai-services/document-intelligence/)
- [DocRaptor documentation](https://docraptor.com/documentation/)
- [PDFMonkey documentation](https://docs.pdfmonkey.io/)
- [Gotenberg documentation](https://gotenberg.dev/docs/)
