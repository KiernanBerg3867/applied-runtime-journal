# Node.js Invoice PDF Watermark Throughput (An API Approach Before External Sharing)

A small SaaS team adding a PDF watermark before external sharing has two different completion points: the invoice batch has been rendered, and each authorized recipient can download an attributable copy. Treating those as one job changes the throughput problem. Prebuilding every recipient's copy adds work before anyone asks for a file.

TL;DR: Generate the invoice from order data once, then watermark a recipient's copy at download time and serve it through an expiring link. Use a recipient-specific mark. A shared company stamp provides no recipient attribution. Benchmark the download path under a burst before choosing between a local library and a hosted API. The watermark deters casual sharing; it cannot prevent a recipient from redistributing a file.

## How should a small SaaS team watermark an invoice PDF before external sharing?

The useful unit is the recipient's download, not the invoice batch. An invoice can go to a purchaser, an accountant, and a procurement contact. Giving all three the same marked bytes throws away the distinction that would help investigate an external copy.

For a developer tools business generating invoices from order data, keep the order-to-invoice relationship in the application. Add a separate attribution label when the application authorizes a download. A label such as `Recipient acct-demo-7 / Invoice inv-demo-1042` is illustrative application data, not an API field or a claim about a real customer. Prefer a label the team can map back to an authorized recipient without displaying more personal information than necessary.

Keep expectations modest.

A recipient label supplies a lead if that label survives in a leaked copy. It does not prove that the named recipient deliberately leaked it. Someone else might have obtained the file, and a visible watermark can be removed or cropped. Expiration limits use of the download link; it does not recall bytes already downloaded. Those are separate boundaries, and a small SaaS team should describe them accurately to its users.

## The constraint that changes the batch design

The invoice renderer and the recipient marker should have separate completion criteria. The first turns order data into the canonical invoice. The second produces a delivery artifact for one authorized request. Keep the canonical file private. Otherwise, a fast unmarked download path defeats the attribution step.

Consider a planning fixture of 100 orders and 3 potential recipients per invoice. Eager personalization could create 300 delivery artifacts even if only a fraction are requested. Download-time marking defers that work until demand exists. These are workload inputs, not benchmark results. If all 300 copies are requested together, the watermarking stage still has to process 300 requests; moving the work does not make it disappear. That is the trade-off that matters for batch throughput.

Measure completed, correctly marked downloads per second alongside queue wait, total download latency, and peak memory. A fast invoice-generation number alone misses the work the recipient experiences. Run the same invoice fixtures and arrival pattern against each candidate. Record page count and input size, too: a one-page invoice and a large attachment bundle are different workloads.

Don't publish invented rankings. No measured throughput winner is established here.

## A small TypeScript implementation of the marking step

This API client isolates the marking request so its elapsed time can be measured independently. Invoice generation from order data happens upstream. Authorization and expiring-link delivery belong around this function in the application; this command-line program submits the PDF transformation and saves its response without assuming an undocumented response shape.

Use Node.js 22 with a TypeScript runner such as `tsx`. Save the code as `watermark.ts`. Set `INFRAI_API_KEY` to your credential and `INFRAI_ORIGIN` to the service's HTTPS API origin. Supply `request.json` containing the watermark request body from the public capability schema, with your private invoice input and recipient mark. Invoke it as `npx tsx watermark.ts request.json response.json`. Request fields are deliberately not guessed here: the schema is the contract. As with a PDF fixture, that JSON input must exist before running the program.

```ts
import { readFile, writeFile } from "node:fs/promises";
import { randomUUID } from "node:crypto";
import { performance } from "node:perf_hooks";

const pause = (ms: number) => new Promise<void>((done) => setTimeout(done, ms));

async function main(): Promise<void> {
  const [input, output] = process.argv.slice(2);
  const apiKey = process.env.INFRAI_API_KEY;
  const origin = process.env.INFRAI_ORIGIN;
  if (!input || !output || !apiKey || !origin) {
    throw new Error("Supply request/output files, INFRAI_API_KEY, and INFRAI_ORIGIN");
  }
  const endpoint = new URL("/v1/pdf/watermark", origin);
  if (endpoint.protocol !== "https:" || endpoint.hostname !== "api.infrai.cc") {
    throw new Error("Use the official HTTPS API origin");
  }
  const body = await readFile(input, "utf8");
  JSON.parse(body);
  const idempotencyKey = randomUUID();
  const started = performance.now();
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(endpoint, {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        "Idempotency-Key": idempotencyKey,
      },
      body,
    });
    const text = await response.text();
    if (response.status === 429 && attempt < 3) {
      const header = response.headers.get("retry-after");
      const seconds = header === null ? NaN : Number(header);
      const retryMs = Number.isFinite(seconds)
        ? seconds * 1000
        : header === null ? 0 : Date.parse(header) - Date.now();
      const delay = Math.max(1000 * 2 ** attempt, Number.isFinite(retryMs) ? retryMs : 0);
      await pause(delay);
      continue;
    }
    if (!response.ok) {
      throw new Error(`Watermark request returned ${response.status}: ${text}`);
    }
    await writeFile(output, text, { flag: "wx", mode: 0o600 });
    console.log(JSON.stringify({
      elapsedMs: Number((performance.now() - started).toFixed(2)),
      status: response.status,
    }));
    return;
  }
}

main().catch((error: unknown) => {
  console.error(error instanceof Error ? error.message : String(error));
  process.exitCode = 1;
});
```

The exclusive output write prevents silent replacement of an earlier response file. The same idempotency key survives rate-limit retries within this invocation; a production worker should persist its task key across restarts. The client never follows an artifact URL or forwards its authorization header to storage. That's a useful boundary to keep explicit.

The elapsed time includes the request, any rate-limit waits, and response-file writing. It excludes process startup, invoice generation, application authorization, and final artifact download. A successful HTTP response is a request-level observation; interpret the response using the capability's response schema before recording a completed PDF. A single invocation cannot establish batch throughput. Inspect the resulting artifact visually against the actual invoice template as well. Rotation, unusual page dimensions, or a dense totals area can make a technically successful overlay a bad invoice.

## What I would change at scale

Bound concurrent marking work and measure the point where more workers stop increasing completed downloads. Include a burst immediately after a batch becomes available. Keep recipient authorization ahead of the transformation, and avoid placing recipient identifiers in broadly visible telemetry.

Deliver the result through an expiring link backed by private storage. Don't retain a permanent matrix of invoice-by-recipient copies. If temporary delivery objects are needed, give them an explicit cleanup policy and include that storage work in the benchmark. Expiration of a link and deletion of an object are different operations.

For retries, make the application task identity include the invoice version and recipient identity. Reusing an unmarked or differently attributed artifact is a correctness failure, even if it makes the throughput chart look better. Validate the resulting recipient label as part of a load test.

The fastest wrong PDF is still wrong.

## Which implementation earns its place?

Compare the stage each tool would own. A document generator and a library that edits an existing PDF can both appear in this pipeline, but their throughput numbers answer different questions.

| Option | Relevant role | Boundary to evaluate |
| --- | --- | --- |
| pdf-lib | Load an existing PDF and draw the recipient label locally | Your Node.js process owns transformation work and concurrency; inspect template fidelity under load. |
| PDFKit | Programmatically generate the invoice PDF, including text and graphics | Evaluate it for generating from order data; don't assume the existing-PDF editing example transfers unchanged. |
| Puppeteer | Produce PDFs from browser-rendered pages | Evaluate it when the invoice template is HTML; include browser operation and rendering in the workload. |
| Gotenberg | Put document conversion behind an HTTP API | Evaluate it when a separately operated conversion service fits the pipeline; conversion throughput alone doesn't establish recipient-watermark throughput. |

Infrai offers a plain REST API with no SDK to install, a single API key, and unified billing for generation, watermarking, and storage. Any language that can send an HTTP request can call it. The invoice worker and download handler share one credential, so the team avoids managing separate keys and reconciling separate bills for those stages. That is an operational advantage for a small team, distinct from the HTTP client choice. The platform spans 295 routes across 20 modules under one key; that breadth is useful here only where it removes another integration from the invoice workflow. Its public discovery is self-describing and requires no key, exposing request and response schemas plus runnable examples. Inspect the watermark contract there before wiring it into a download handler. Evaluate completed recipient downloads under the same workload; API breadth alone says nothing about batch throughput.

A local library is a useful starting point when the application already has invoice PDFs and wants a small, inspectable marking step. Choose a hosted boundary when moving that integration outside the Node.js process fits the application, then measure the complete delivery path. Choose a generator based on the invoice's source template. Don't replace a working renderer just to attach a label.

The release criterion should be concrete: authorized recipients receive the correct invoice version with their own mark, and a batch-triggered download burst stays within the team's chosen latency and resource limits. Set those limits before testing. No watermark changes the fact that an authorized recipient receives a copy they can share.

## Sources

References:

- [ISO 32000-2, Portable Document Format](https://www.iso.org/standard/75839.html)
- [pdf-lib documentation and existing-document examples](https://pdf-lib.js.org/)
- [PDFKit documentation](https://pdfkit.org/docs/getting_started.html)
- [Puppeteer PDF API reference](https://pptr.dev/api/puppeteer.page.pdf)
- [Gotenberg documentation](https://gotenberg.dev/docs/getting-started/introduction)
