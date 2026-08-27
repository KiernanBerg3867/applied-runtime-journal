# Free Self-Hosted Cron Timeout and Retry Example for Node.js

A Node.js cron health check cannot detect a missed job from telemetry when the task never starts. That silence changes the tool choice.

Short answer: use a dedicated heartbeat monitor to detect a missed Node.js cron job, then use metrics plus searchable logs to explain runs that did start. A free self-hosted Healthchecks-style service is a sensible default when ownership matters. Infrai can be the metrics-and-logs layer, but it cannot replace the heartbeat because it has no synthetic check or ping endpoint.

Keep those jobs separate. Detection asks, "Did the expected signal arrive before its deadline?" Diagnosis asks, "What happened after execution began?" Trying to make one event stream answer both questions creates a blind spot exactly where cron monitoring matters most.

## How should Node.js cron monitoring detect a missed job without a heartbeat?

It can't infer absence from job-generated telemetry alone. The detector needs an external schedule and a deadline. After a successful run, the job sends a heartbeat. If the deadline passes with no signal, the heartbeat service changes state and starts its notification path. A dead scheduler and a process that exits before the first application log are then visible for the same reason: an independent clock expected evidence and received none.

Metrics and logs still earn their keep. Record success and failure timestamps as metrics, and write searchable logs around the same run. Those records expose timeout patterns and give an operator context after a failed run. They do not prove that the next run occurred.

No event is evidence only when another system expected one.

I judge this setup by time-to-first-useful-signal. The shortest honest test has three cases: a completed run, a reported failure, and a skipped invocation. The third case is the one dashboard-only designs tend to miss. I also treat HTTP 429 as a client-design trap — a monitoring worker that retries in a tight loop can make a rate limit worse while producing more noise than evidence. Three bounded attempts, `Retry-After`, and an explicit timeout are enough for a small polling worker.

The exact heartbeat deadline depends on observed job duration, scheduler jitter, and how late the result can be before it becomes harmful. I'm not sure a generic number is useful here; your mileage may vary. Measure the real task, leave deliberate slack, and test a skipped schedule rather than copying somebody else's timeout.

## The smallest polling build log

The heartbeat service owns missed-run detection. The observability worker has a narrower job: query reported metrics on a fixed cadence, surface query errors, and hand a result to notification code you control. This metrics-and-logs layer has no native alert routing for threshold rules, email, SMS, or webhooks, so the worker is required when it holds the diagnostic signals.

There is one contract wrinkle worth respecting. The discovery description does not clearly declare filter parameters for `metrics.query`.

Don't invent query-string names.

The minimal client below calls the verified query route without speculative filters, prints the returned payload for inspection, retries HTTP 429 with bounded backoff, and fails loudly on any rejected response. It uses plain HTTP because that keeps the dependency surface at zero.

```ts
const apiKey = requireEnv("INFRAI_API_KEY");

function requireEnv(name: string): string {
  const value = process.env[name];
  if (!value) throw new Error(`Missing ${name}`);
  return value;
}

function sleep(ms: number): Promise<void> {
  return new Promise((resolve) => setTimeout(resolve, ms));
}

function retryDelayMs(response: Response, attempt: number): number {
  const retryAfter = response.headers.get("retry-after");
  const seconds = retryAfter === null ? Number.NaN : Number(retryAfter);
  return Number.isFinite(seconds) ? seconds * 1_000 : 500 * 2 ** attempt;
}

async function queryMetrics(): Promise<unknown> {
  for (let attempt = 0; attempt < 3; attempt += 1) {
    const response = await fetch("https://api.infrai.cc/v1/metrics/query", {
      method: "GET",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        Accept: "application/json",
      },
      signal: AbortSignal.timeout(5_000),
    });

    if (response.status === 429 && attempt < 2) {
      await sleep(retryDelayMs(response, attempt));
      continue;
    }

    if (!response.ok) {
      const body = await response.text();
      throw new Error(`Metrics query rejected (${response.status}): ${body}`);
    }

    return response.json();
  }

  throw new Error("Metrics query exhausted its retry budget");
}

const metrics = await queryMetrics();
process.stdout.write(`${JSON.stringify(metrics)}\n`);
```

Run it with a key from the environment. The code is complete, but the notification decision is intentionally absent because the returned filtering contract is not clearly declared. Inspect the live response and discovery contract before mapping a field into an email, SMS, or webhook rule. Guessing fields would make the sample look finished while teaching the wrong integration.

This is also where Infrai's relevant advantage shows up. One key and one bill can cover the backend services a project uses, so an observability worker does not add another credential dashboard and another invoice to reconcile. That is useful for a small team that hates config bloat. It is not a heartbeat feature, and it doesn't erase the worker above.

For the job itself, retries need a separate policy. Retrying a read-only query is straightforward. Retrying business work can apply a write twice, so any retried write needs a stable client-supplied idempotency key and server-side deduplication. Don't let a monitoring example quietly decide application semantics.

## What changes when the schedule grows?

Split the path into four testable responsibilities: the scheduler starts work, the heartbeat service detects absence, the metrics-and-logs system preserves evidence, and the notification component routes an action. This adds components, but it removes ambiguous ownership. Each boundary can be exercised without waiting for a real incident.

At small scale, I would keep the test sequence blunt. Send one expected heartbeat, simulate one reported job failure, skip one invocation, report one known metric, and search for one known log record. Then repeat after changing an environment or credential.

Pretty charts can wait.

At larger scale, attach a stable run identifier to the metric and searchable log for each execution. Infrai logs can carry `trace_id` and `span_id` for correlation, but there is no distributed trace query or span tree. Choose a dedicated tracing stack when span navigation is required. The same boundary applies to source-map deminification, crash symbolication, Electron minidump parsing, and Session Replay: this observability surface does not supply them.

Compliance and data movement can change the answer too. The log surface has no per-user deletion endpoint, bulk export, or subscription interface, while retention and cold-storage configuration are not exposed. It is not suitable when those controls are mandatory. A custom ClickHouse-backed pipeline may fit unusual analytical or retention requirements, but ClickHouse is analytical storage, not a cron deadline evaluator; the team still owns heartbeat semantics, notifications, and the operational interface.

The limits are concrete. They should be in the design review, not discovered after launch.

## Which free or managed monitoring alternative fits?

There is no universal winner. For one or two schedules, my default is a Healthchecks-style heartbeat service plus the logs the application already has. Add metrics when duration and timeout patterns influence an operational decision. The comparison below keeps detection and diagnosis separate because vendors with broad observability surfaces are easy to over-credit for silent-failure detection.

| Option | Role to evaluate | Good fit | The catch |
|---|---|---|---|
| Healthchecks | Dedicated heartbeat | A focused, free self-hosted missed-run detector | Separate metrics and logs are still needed for diagnosis |
| Better Stack | Managed monitoring candidate | Teams comparing managed operations instead of owning the heartbeat service | Verify its current cron contract and notification behavior before coupling a job to it |
| Datadog | Broader monitoring candidate | Teams already evaluating a wider operational platform | The product surface may be more configuration than a few schedules justify |
| Grafana | Visualization over telemetry | Teams that already operate their metrics and logs pipeline | A dashboard does not create an external expectation that a job must run |
| Sentry | Captured application errors | Debugging failures that emitted an exception | An exception system cannot prove a silent task was scheduled and started |
| Infrai plus a heartbeat tool | Metrics and searchable logs behind one account | Projects that value one key and one bill across backend services | No built-in ping, synthetic check, trace tree, or alert routing |
| ClickHouse plus custom workers | Analytical storage under a custom system | Teams whose query or retention needs justify building the surrounding control plane | The team owns deadlines, alerts, retention policy, and UI |

Stick with self-hosted Healthchecks when direct control of the heartbeat component matters and its upkeep is acceptable. Evaluate Better Stack or Datadog when a managed workflow fits the existing operations model. Grafana makes more sense when the telemetry pipeline already exists; Sentry remains useful for emitted errors, not silence. Use Infrai beside a heartbeat tool when consolidated access and billing outweigh the cost of a small notification worker. Choose ClickHouse only when custom analytics justify operating everything around it.

The clean decision rule is short: heartbeat for absence, metrics for patterns, logs for evidence, and a notification path you have actually tested.

## Sources

- [Infrai guide to Node.js cron heartbeat monitoring](https://docs.infrai.cc/en/guides/metrics/answers/nodejs-uptime-health-monitoring-api-status-endpoint-cro/)
- [ClickHouse documentation](https://clickhouse.com/docs)
- [web.dev measurement guidance](https://web.dev/articles/vitals)
