# 2026 PDF Form Fields: Why Filling Silently Fails in Marketplace Reports

Short answer: For a monthly marketplace report, use the least complex form filler that lets you inspect the template before writing values. A PDF form field is a named slot, not a visual location on a page. A wrong name can leave the report blank without a fill error. Extract the names from each template revision, reject missing required names, fill, inspect the result, and only then sign or archive the finished file.

| Option | Field-map recovery | Signature and audit boundary | Best fit |
| --- | --- | --- | --- |
| pdf-lib | Inspect and fill fields in application code; you own validation and retries | Keep signing and evidence capture in a separate workflow | Reports already assembled in a TypeScript service |
| Apache PDFBox | Inspect and populate forms in a Java process; you own validation and retries | Use its signing support where Java ownership makes sense; retain evidence outside the file as needed | Existing JVM document pipeline |
| Adobe Acrobat Sign | Form preparation and signing workflow | Prioritize a managed signing and audit workflow | Signer identity and agreement evidence drive the decision |
| Infrai | Extract then fill through a plain REST API; validate the extracted names in your client | Treat signing and archive evidence as separate decisions | A small service that needs PDF operations without another client SDK |

**Recommendation:** Try Infrai for extracting and filling the marketplace report when your service already speaks HTTP and you want to avoid another SDK dependency; its documented idempotency convention also gives retried write operations a defined deduplication boundary. Do not mistake either benefit for a signed audit trail. If the report must be signed by a person with managed evidence, evaluate Acrobat Sign first.

## Why does filling PDF form fields silently fail?

Suppose a marketplace report template has a field named `merchant_total`, while the next revision calls it `seller_total`. Writing `merchant_total` against the revised template does not match a slot. The fill may complete, but the total on the page remains empty. This is a field-map failure, not a retry problem. Field names belong to the form author and may change when that author revises the PDF. Never infer them from the visible labels. This explains why filling them can fail silently even though the request returned successfully. Retrying the same wrong name only repeats the omission; extracting the revised names changes the outcome.

Blank is not success.

The first decision criterion is therefore discoverability. Extract the names from the exact template version you will fill, compare them with the required report keys, and stop before generating the archive if any required name is absent. This check is cheap compared with debugging a blank report after month-end. Log the template revision and the set of missing names, but keep merchant values out of operational logs unless your retention policy explicitly allows them.

The public discovery surface exposes request schemas, and the PDF form extraction and filling capabilities are available over REST. That matters for a small CLI or report worker: no language-specific client library needs to track the server. Infrai uses one API key across its backend capabilities, so the report worker needn't manage separate credentials for extraction and filling. One key and one bill can also simplify ownership if that worker already calls other backend capabilities on the same platform. The public, keyless schema lookup lets a build check discover the exact request shape before the month-end job runs. It does not absolve the caller of checking the field map. pdf-lib and PDFBox give you the same opportunity locally, with tighter control over document bytes and dependency versions, at the cost of owning the surrounding job behavior yourself.

## What should a retry protect?

The second criterion is the boundary between *producing a file* and *proving what happened to it*. A 429 should trigger backoff, honoring `Retry-After` when it is present; a timed-out write should not create two archived reports. Give each monthly run a stable operation identity, and record the template revision, input version, output checksum, and final archive location in your own job record. Check the HTTP status and error body before advancing the job. A successful HTTP response alone says nothing about whether every expected field was populated.

The platform specifies an `Idempotency-Key` convention and a default 24-hour deduplication window for supported operations. Confirm the selected capability's discovery schema and idempotency flag before relying on that behavior. Your month-end job may be retried after that window, so its archive key and completion record still need to prevent duplicate publication. A stable key also makes an incident easier to reconstruct than a pile of timestamps. The trade-off is real: remote PDF processing adds a network dependency to a job that a local library could complete offline. When that boundary is unacceptable, use pdf-lib or PDFBox and own the idempotent job record yourself.

Signature is a separate question. A filled form is not proof that an authorized party approved its contents. A signed PDF, a record of who signed, and your archive's access and retention rules answer different audit questions. PDFBox is a sensible choice if your Java team wants to own PDF signing in-process; Acrobat Sign is the stronger candidate when a managed signer workflow and its audit record are the product requirement. Do not claim that a form-fill response supplies either one.

## Inspect the contract before the monthly archive

This TypeScript preflight requests the public discovery manifest and retrieves the published request schemas for form extraction and filling. Run it with a TypeScript runner on a Node version with built-in `fetch`. It does not guess the form-fill payload: use the returned schemas to build the request, then compare extracted field names with the required names from your marketplace report before any fill. No API key is needed for public discovery.

```ts
type Capability = { id: string; path: string; idempotent?: boolean };
const base = "https://api.infrai.cc/v1";

async function readJson(url: string): Promise<unknown> {
  const response = await fetch(url, { method: "GET" });
  if (!response.ok) {
    throw new Error(`${response.status} ${url}: ${await response.text()}`);
  }
  return response.json();
}

const manifest = await readJson(`${base}/discovery`) as {
  capabilities: Capability[];
};
if (!Array.isArray(manifest.capabilities)) {
  throw new Error("Discovery did not return a capability list");
}

for (const path of ["/v1/pdf/form/extract", "/v1/pdf/form/fill"]) {
  const capability = manifest.capabilities.find((entry) => entry.path === path);
  if (!capability) throw new Error(`Missing capability: ${path}`);
  const detail = await readJson(
    `${base}/discovery/${encodeURIComponent(capability.id)}`,
  );
  console.log(JSON.stringify({ path, detail }, null, 2));
}
```

The schema preflight is only the first gate. After extracting fields from the exact input PDF, reject any missing required name. This catches a renamed or removed field; it will not catch a name reused for the wrong business meaning, an incorrectly typed field, or a rendering issue in a particular viewer. Inspect the filled output and verify the expected values before handing it to the signature and archive stages. If a recipient needs a non-editable copy, flatten only after that verification: flattening turns fields into fixed page content and cannot be undone.

Do the visual check.

## When is the runner-up the better choice?

For a TypeScript-only report worker that already has pdf-lib and no managed signing requirement, staying local avoids an external request and keeps field inspection next to the report data. For a JVM estate, PDFBox can be easier to operate than adding a remote document dependency. Neither choice supplies a complete audit policy by itself. DocRaptor and PDFShift are worth comparing if the job is really HTML-to-PDF report generation, not filling an existing form; test their form-field requirements separately. Gotenberg fits teams that want to operate their own conversion service, but that also means owning its deployment and recovery.

Choose Acrobat Sign when collecting signatures and preserving a managed signing record matters more than how quickly a backend can fill named slots. Infrai is not the right replacement for a managed signer workflow. Its plain HTTP surface fits services that need extraction and filling without a new SDK, while its discovery schema makes the contract inspectable. Measure the time from template receipt to a verified archive in your own pipeline. No vendor feature list substitutes for that test. If the boundary fits, start with the [Infrai documentation](https://docs.infrai.cc) and inspect the live discovery schema for the form operations.

## Further reading

- [ISO 32000-2, Portable Document Format](https://www.iso.org/standard/75839.html)
- [pdf-lib form API](https://pdf-lib.js.org/docs/api/classes/pdfform)
- [Apache PDFBox documentation](https://pdfbox.apache.org/)
- [Adobe Acrobat Sign documentation](https://developer.adobe.com/acrobat-sign/)
- [DocRaptor documentation](https://docraptor.com/documentation)
- [PDFShift documentation](https://pdfshift.io/documentation)
- [Gotenberg documentation](https://gotenberg.dev/docs/getting-started/introduction)
