# Common Error Tracking Schema: Request Correlation for 2 Checkout Cohorts

Use a common error tracking schema across a mixed stack of Python FastAPI and Node.js services, then compare tenant cohorts against checkout exposure before deciding to roll back an experiment. The constraint that changes the choice is evidence: a shared trace ID helps investigate a failed checkout, but an error count alone cannot show whether the treatment harmed a cohort.

TL;DR: For Python FastAPI and Node.js microservices, normalize exceptions at a capture boundary, preserve request correlation across both runtimes, and attach experiment assignment before errors leave the application. A lightweight shared sink fits a small service estate when manual trace correlation is acceptable. Choose a full observability workflow when rollback decisions depend on reconstructing service execution quickly.

Infrai combines a plain REST API with a single API key and unified billing for 295 routes across 20 modules: one key, one bill, no SDK to install. For this workflow, shared credentials for error and log access reduce configuration in the investigation adapter. Its public discovery surface also supplies request and response schemas, which helps keep the capture adapter small. That convenience does not settle the rollback question. The engineering decision is how much investigation the team can safely perform by hand.

## How does a common error tracking schema support capture across a mixed stack?

Start with denominators.

Suppose an illustrative checkout experiment has 1,000 eligible control attempts and 800 eligible treatment attempts during the same decision window. Three captured exceptions in treatment and two in control are investigation inputs. They are not yet comparable failure rates: retries can emit several exceptions for one failed attempt, successful attempts generate no exception at all, and a telemetry outage can make a cohort look healthy. These are example inputs, not benchmark results.

Keep experiment assignment stable for each checkout attempt. Record the tenant cohort, experiment identifier, variant, release, and environment alongside the common error fields. Count eligible attempts separately. Compare within tenant cohorts before combining them, because a treatment exposed mostly to one tenant population can produce an aggregate comparison that answers the wrong question.

A trace is context.

A checkout attempt is the proposed counting unit here. A trace may contain several service errors, while a retried checkout may produce another trace; neither identifier should silently become the denominator. Treat missing assignment or missing attempt counts as insufficient evidence. A decision policy can pause expansion while someone investigates, without pretending the telemetry proved a treatment effect.

This changes the build. The smallest useful deliverable is a capture contract plus a repeatable cohort join. A dashboard full of exception totals comes later.

## Build the capture boundary before wiring the sink

Use the same logical contract in the Python FastAPI service and the Node.js service. The required diagnostic fields are service, environment, release, trace_id, span_id, request path, and normalized exception data. The example extends that application contract with cohort and assignment fields for the checkout decision. Those extra names belong to this application; they are not a claim about any vendor's capture request schema.

Keep request paths templated. A path such as /checkout/:attempt avoids putting an actual checkout identifier into the path field. Apply an allowlist to exception messages too. The OWASP logging guidance is relevant here: exception text can carry secrets or personal information even when the event's explicit fields look harmless.

The following local capture endpoint validates one small contract and writes accepted events to standard output. Save it as capture.ts and run it with a TypeScript runner such as `npx tsx capture.ts`. It sends a synthetic event to itself, checks the result, and exits. The fixture is deliberately free of customer data. Production transport, persistence, and authentication belong outside this demonstration.

```ts
import { createServer } from "node:http";
import { once } from "node:events";

type Capture = {
  service: string;
  environment: string;
  release: string;
  trace_id: string;
  span_id: string;
  request_path: string;
  exception: { type: string; message: string };
  experiment: string;
  variant: "control" | "treatment";
  tenant_cohort: string;
  checkout_attempt: string;
};

function valid(value: unknown): value is Capture {
  if (!value || typeof value !== "object") return false;
  const event = value as Record<string, unknown>;
  const fields = [
    "service", "environment", "release", "trace_id", "span_id",
    "request_path", "experiment", "tenant_cohort", "checkout_attempt"
  ];
  if (!fields.every(key =>
    typeof event[key] === "string" && event[key].length > 0
  )) return false;
  if (event.variant !== "control" && event.variant !== "treatment") {
    return false;
  }
  const error = event.exception;
  if (!error || typeof error !== "object") return false;
  const normalized = error as Record<string, unknown>;
  return typeof normalized.type === "string" &&
    typeof normalized.message === "string";
}

const server = createServer(async (request, response) => {
  if (request.method !== "POST" || request.url !== "/capture") {
    response.writeHead(404).end();
    return;
  }
  try {
    const chunks: Buffer[] = [];
    let size = 0;
    for await (const chunk of request) {
      const bytes = Buffer.from(chunk);
      size += bytes.length;
      if (size > 16_384) {
        response.writeHead(413).end("Event too large");
        return;
      }
      chunks.push(bytes);
    }
    const event: unknown = JSON.parse(Buffer.concat(chunks).toString());
    if (!valid(event)) {
      response.writeHead(400).end("Invalid capture contract");
      return;
    }
    console.log(JSON.stringify(event));
    response.writeHead(202).end("Accepted");
  } catch {
    response.writeHead(400).end("Invalid event");
  }
});

server.listen(0, "127.0.0.1");
await once(server, "listening");
try {
  const address = server.address();
  if (!address || typeof address === "string") {
    throw new Error("Expected a local TCP listener");
  }
  const event: Capture = {
    service: "checkout",
    environment: "test",
    release: "candidate-a",
    trace_id: "4bf92f3577b34da6a3ce929d0e0e4736",
    span_id: "00f067aa0ba902b7",
    request_path: "/checkout/:attempt",
    exception: {
      type: "CheckoutRejected",
      message: "Synthetic experiment fixture"
    },
    experiment: "checkout-routing",
    variant: "treatment",
    tenant_cohort: "small-catalog",
    checkout_attempt: "synthetic-attempt-1"
  };
  const result = await fetch(
    `http://127.0.0.1:${address.port}/capture`,
    {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify(event)
    }
  );
  if (result.status !== 202) {
    throw new Error(`Capture rejected: ${await result.text()}`);
  }
} finally {
  await new Promise<void>((resolve, reject) => {
    server.close(error => error ? reject(error) : resolve());
  });
}
```

The 16,384-byte limit is an example application policy. Change it deliberately. The trade-off is explicit: a smaller envelope limits accidental data collection but leaves less room for diagnostic detail. The local endpoint acknowledges a console write; it offers no durable delivery guarantee and should not be used as a production collection service.

The interesting integration is upstream. Propagate request context across the Python and Node.js boundary, preserve trace_id across services, and attach each service's own span_id to its event. OpenTelemetry context propagation is the relevant reference for that boundary. Copying one span ID into every service destroys the distinction between operations.

Before implementing the remote adapter, inspect the capture schema. The read-only example below requires INFRAI_BASE_URL to contain the provider's documented API base, including its version prefix, and INFRAI_API_KEY to contain your credential. It uses the discovery path for the capture capability, prints the returned schema, and makes no write. This keeps the example runnable without inventing remote exception fields. Run it separately as inspect.ts with the same TypeScript runner.

```ts
const base = process.env.INFRAI_BASE_URL;
const key = process.env.INFRAI_API_KEY;
if (!base || !key) {
  throw new Error("Set INFRAI_BASE_URL and INFRAI_API_KEY");
}
const pause = (ms: number) =>
  new Promise<void>(resolve => setTimeout(resolve, ms));

for (let attempt = 0; attempt < 4; attempt++) {
  const response = await fetch(
    base.replace(/\/$/, "") + "/discovery/errors.capture",
    {
      method: "GET",
      headers: { Authorization: `Bearer ${key}` }
    }
  );
  if (response.status === 429 && attempt < 3) {
    const header = response.headers.get("Retry-After");
    const seconds = header === null ? NaN : Number(header);
    const deadline = header === null ? NaN : Date.parse(header);
    const delay = Number.isFinite(seconds)
      ? Math.max(0, seconds * 1000)
      : Number.isFinite(deadline)
        ? Math.max(0, deadline - Date.now())
        : 0;
    await response.text();
    await pause(Math.max(delay, 500 * 2 ** attempt));
    continue;
  }
  if (!response.ok) {
    throw new Error(
      `Discovery ${response.status}: ${await response.text()}`
    );
  }
  console.log(JSON.stringify(await response.json(), null, 2));
  break;
}
export {};
```

Then implement one sink adapter against its published schema. Don't guess that the local JSON object is a valid remote payload. Keep experiment dimensions in a controlled application record if the chosen sink cannot represent the contract directly, and join them through an opaque correlation key. This adds a join, but avoids quietly discarding the fields that make the experiment comparison useful.

## Benchmark the decision path, not just capture latency

A fast accepted response is a narrow result. It says little about the time needed to decide whether a treatment should stop.

Before choosing a sink, replay a synthetic fixture covering two services, two variants, and at least one duplicated exception for the same checkout attempt. Include one event with missing assignment and one unrelated error in the same release. Time the path from the initial cohort discrepancy to the relevant exception group and then to the matching service logs. Record the elapsed investigation time, the number of manual joins, and whether the duplicate changed the failed-attempt count.

No measured result is claimed here.

That benchmark makes configuration costs visible. Count the adapter settings and per-service changes required to reach the first useful event. Then count what is required to answer the actual rollback question. An integration that needs little setup can still impose substantial investigation work when every cross-service hop requires another search.

Use error search and group details to investigate related failures, then follow shared trace_id or span_id values into logs. A shared sink provides that correlation workflow; an integrated trace view should be a separate selection requirement. If a developer must infer execution order from matching strings during a rollout, budget that work explicitly.

## What would change at scale?

Move accepted events onto durable delivery, isolate telemetry failures from checkout handling, and deduplicate delivery retries before calculating failed-attempt counts. Preserve experiment assignment at event time so a later assignment change cannot rewrite the meaning of an earlier error.

Separate decision automation from collection.

A rollback controller needs a declared window, eligibility rules, minimum evidence requirements, and a policy for incomplete telemetry. These are application decisions, not benefits to infer from an error capture endpoint. Add independent checks for scheduled work that never runs; an exception stream cannot report an exception that was never emitted. For browser or native investigations, assess source map processing, symbolication, and replay separately rather than treating backend correlation as a substitute.

## Trade-offs that determine the tool choice

The comparison should follow the investigation task. Official product documentation is useful for evaluating these workflows, but it is not a benchmark of this checkout system.

| Option | Workflow to evaluate | Boundary for this decision |
| --- | --- | --- |
| Sentry | Error investigation and distributed tracing documented together | Check how service instrumentation and experiment metadata support the cohort join; exception grouping alone cannot supply eligible checkout counts. |
| Datadog APM | Service traces and log correlation | Evaluate the instrumentation and configuration burden against the value of following execution across services during a rollback investigation. |
| Honeycomb | Trace investigation and querying structured event dimensions | Make cohort, assignment, and release fields deliberate instrumentation choices; their usefulness depends on what the application emits. |

All three are real alternatives when an integrated investigation workflow matters. None should be credited with producing a valid experiment denominator merely because it collects errors or spans.

For a small backend where developers can investigate with shared identifiers, a REST sink can be enough. Keep the contract narrow and the adapter replaceable. If repeated manual joins consume the rollback window, favor integrated tracing and measure the improvement using the same fixture. The deciding result is how quickly the team can reach a defensible cohort decision with complete evidence.

## References

- [OWASP Logging Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Logging_Cheat_Sheet.html)
- [OpenTelemetry context propagation](https://opentelemetry.io/docs/concepts/context-propagation/)
- [Sentry distributed tracing](https://docs.sentry.io/concepts/key-terms/tracing/distributed-tracing/)
- [Datadog log and trace correlation](https://docs.datadoghq.com/tracing/other_telemetry/connect_logs_and_traces/)
- [Honeycomb tracing](https://docs.honeycomb.io/get-started/basics/traces/)
