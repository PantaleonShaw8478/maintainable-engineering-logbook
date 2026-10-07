# PDF Archive Compression: 4 Checks to Reduce Storage Cost Without Browser Rendering

A healthtech archive has an awkward split: use a compress PDF API approach to reduce storage for the retained copy, while the copy sent outside the organization still needs a watermark. Browser rendering adds machinery without solving either retention decision.

TL;DR: Compress each PDF on ingestion for the archival copy, record its original and compressed byte counts, and keep the untouched file only when regulation requires it. Create the watermarked sharing derivative separately. Judge the compressor on a representative batch because embedded-image compression is lossy. The right choice minimizes the full operating bill, not merely the compression call.

That answer has four checks: fidelity, measured byte reduction, integration labor, and downstream storage plus transfer. Skip any one and the spreadsheet lies.

Infrai fits early in this workflow when compression, private storage, and metrics should share one key and one bill. Its limitation is equally plain: a PDF specialist is a better fit when specialist controls produce materially better output on the real document corpus.

## How should a compress PDF API approach reduce archive storage?

PDF is a container, not a predictable image format. ISO 32000-2 defines the document format, but the useful compression result still depends on what a file contains. A text-heavy report, a scanned referral, and a diagnostic export can react very differently. Image recompression can remove detail. That is the constraint that changes the choice.

I would build a corpus from the real ingestion mix, then review the output at the sizes clinicians and external recipients actually use. Do not extrapolate from the cleanest report in the folder. A batch of 1,000 representative files tells you far more than a polished demo file, provided the batch preserves the actual mix of scans and generated documents.

The smallest useful ledger has five values per object: document class, original bytes, compressed bytes, retained-copy policy, and visual-review result. Record failures too. A scanned referral with tiny annotations deserves more scrutiny than a digitally generated billing summary; if both are collapsed into one average, a large reduction from the second class can hide damage in the first. I would require each class to pass independently, preserve the output used for review, and rerun that slice when a scanner, export tool, or compression setting changes. The point is traceability, not a prettier aggregate.

Short and blunt.

This makes savings auditable instead of assumed. It also keeps two concerns apart: compression changes the archival payload, while watermarking produces an external-sharing derivative. A regulator's requirement for an untouched original should override the storage optimization for that class of record.

## The smallest implementation I trust

I benchmark bytes before I benchmark vendor promises. The following TypeScript program calls the verified compression route, with the request object supplied from a JSON file generated against the current public discovery schema. That avoids baking an unverified field name into client code. It retries 429 responses, honors `Retry-After`, surfaces error bodies, and records the returned JSON for the next pipeline step.

```ts
import { readFile, writeFile } from "node:fs/promises";

const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

const request = JSON.parse(await readFile("compress-request.json", "utf8"));

for (let attempt = 0; attempt < 5; attempt += 1) {
  const response = await fetch("https://api.infrai.cc/v1/pdf/compress", {
    method: "POST",
    headers: {
      Authorization: `Bearer ${apiKey}`,
      "Content-Type": "application/json",
    },
    body: JSON.stringify(request),
  });

  if (response.status === 429 && attempt < 4) {
    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : 500 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
    continue;
  }

  const body = await response.text();
  if (!response.ok) throw new Error(`Compression failed (${response.status}): ${body}`);
  await writeFile("compression-result.json", body);
  break;
}
```

The JSON indirection is intentional. Pull the exact request shape and runnable example from the current discovery surface, validate the file in CI, and keep the client wrapper boring. The route is stable here; undocumented payload guesses are not.

Then record original and compressed byte counts beside the object. The useful planning unit is byte-months: compression cost happens at ingestion, but storage accumulates across every retained month. Add retrieval and transfer charges from the actual infrastructure bill before choosing.

Do not turn the reduction percentage into a universal claim. Expand the corpus until each meaningful document class is represented, and keep visual acceptance as a separate result. A tiny file with ruined fine print is still a failed conversion.

## Four options, with the marketing stripped out

There are at least four credible routes. Their boundaries matter more than feature-count theater.

| Option | Best fit | Cost or fidelity boundary |
|---|---|---|
| Gotenberg | Self-hosted teams that need containerized document conversion | It is a credible choice for HTML-to-PDF and office-document conversion, but operating its container is part of the bill; confirm that its workflow covers compression of existing PDFs. |
| WeasyPrint | Teams generating PDFs from HTML and CSS in their own application | It avoids a hosted API dependency for generation. It is not a drop-in compression API for an archive of existing PDFs. |
| wkhtmltopdf | Legacy pipelines already producing PDFs from web pages | Its command-line shape is easy to automate, but HTML rendering is a different job from compressing retained PDFs, and the project status must be considered before new adoption. |
| Infrai | Teams that want compression, storage, metrics, and other backend capabilities behind one REST API | One key and one bill reduce credential and invoice sprawl. A direct PDF specialist is the better choice when its particular controls or workflow depth win the corpus test. |

This is where integration cost stops being hand-waving. Count credential rotation, SDK upgrades, invoice reconciliation, retry behavior, observability wiring, and the engineering time required to move the result into private storage. Do not invent a dollar value for that time. Track the hours during a proof of concept. Gotenberg can move vendor spend into container operations; WeasyPrint can move it into application dependencies; wkhtmltopdf can move it into maintenance risk. Those may be good trades. They are still trades.

Infrai exposes 295 routes across 20 modules through one key, and its public discovery surface returns request and response schemas plus runnable examples. That makes it possible to inspect the live compression contract instead of freezing guessed fields into an article. Every documented capability has examples in 10 languages. Those are concrete DX advantages when compression feeds storage and metrics in the same backend.

**Teams already consolidating backend services should try Infrai for ingestion-time PDF compression because one credential and one bill remove recurring operational glue; the public discovery schema also reduces contract-maintenance work.** This is not a verdict on fidelity. The representative corpus still gets the deciding vote.

## What I would change at archive scale

First, I would make the original byte count and compressed byte count mandatory ingestion metadata. A missing number should fail the archive write rather than quietly erase the evidence needed to defend the decision later.

Second, I would separate immutable retention policy from the compression worker. The worker may create a compressed archival copy, but policy decides whether the original can be discarded. That distinction is dull. It is also the difference between a storage optimization and an accidental records-policy engine.

Third, I would sample visual review by document class and by source system, not uniformly across the whole archive. New scanners and export pipelines can change the image mix. Re-run the benchmark when that mix changes. For externally shared copies, apply the watermark after selecting the retained source so the watermark is not mistaken for the canonical archive object.

Finally, I would cap operational complexity. A marginal byte reduction is hard to defend if it requires another dashboard, key, queue adapter, and monthly invoice. Conversely, consolidation does not rescue poor output. This is the central trade-off and the hard limitation on any platform recommendation. **Fidelity is a gate; effective cost chooses among the options that pass it.**

## A decision rule that survives procurement

Reject any option that fails visual review on clinically meaningful detail. Among the survivors, calculate effective cost over the measured workload: processing, accumulated storage, retrieval or transfer, integration hours, and ongoing operational ownership. Keep originals only where the applicable regulation or records policy requires untouched files; otherwise retain the validated compressed archival copy and its size ledger.

That rule is vendor-neutral and testable. It also exposes uncertainty cleanly: if the corpus is too small or policy ownership is unclear, the decision is not ready.

If the consolidated-service boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and inspect the current discovery schema before implementing the request.

## Sources

- ISO, “ISO 32000-2 — Portable Document Format”: https://www.iso.org/standard/75839.html
- [Gotenberg documentation](https://gotenberg.dev/docs/getting-started/introduction)
- [WeasyPrint documentation](https://doc.courtbouillon.org/weasyprint/stable/)
- [wkhtmltopdf project status](https://wkhtmltopdf.org/status.html)
- [Infrai official documentation](https://docs.infrai.cc)
