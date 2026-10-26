# Handling Moderation API Timeout Errors with Nodejs Retry Logic Explained

Moderation timeouts are a cost-accounting problem before they are a retry problem. In an edtech knowledge-base service, I would retry a bounded set of transient failures, but I would record every attempt against the tenant before sending it. A timeout says the client lacks a result. It does not prove that the remote service did no work.

Short answer: set a deadline with `AbortController`, retry at most twice with jitter for network timeouts and retryable HTTP responses, and mark timed-out attempts as `unknown` in the tenant ledger. Do not turn an unknown moderation result into an automatic approval. Queue it, reject it, or route it to a policy-approved fallback.

| Choice | Retry policy | Tenant cost treatment | Best fit |
| --- | --- | --- | --- |
| Synchronous retry | Two retries, exponential backoff with jitter | Count each started attempt; reconcile unknowns later | Interactive search with a short user deadline |
| Deferred review | Stop after the first timeout and enqueue the item | Keep the attempt open until its final state | High-risk exam or student-submitted content |
| Fail closed | Return a moderation failure to the caller | Record one attempt and the failure reason | Content that must not be shown without a decision |

**My default is synchronous retry for low-risk knowledge-base questions, followed by a deferred state when the deadline expires.** It keeps the first response useful without pretending that a timeout is free. The rest of the design is about making that choice observable per tenant.

## What Should a Moderation Timeout Count As?

A timeout has two separate meanings. Locally, the request exceeded its deadline. Remotely, the outcome is unknown. Those statements can both be true even if the HTTP request never produced a response.

Keep the states separate.

That distinction matters for a private knowledge base serving several schools. A request can contain retrieved text, a student question, or a teacher's correction. If the moderation decision is unknown, the application should not silently treat the content as clean. The caller needs a stable state such as `pending`, `blocked`, or `retryable_error`, rather than a Boolean assembled from missing data.

The accounting record should be created before the request begins. A small event is enough:

```ts
type ModerationAttemptState =
  | "started"
  | "approved"
  | "rejected"
  | "unknown"
  | "failed";

type ModerationAttempt = {
  tenantId: string;
  requestId: string;
  attempt: number;
  state: ModerationAttemptState;
  startedAt: string;
  finishedAt?: string;
  reason?: string;
};
```

The ledger is not a prediction of a provider's bill. It is an internal usage trail. That is the useful contract: finance and platform teams can group attempts by `tenantId`, while the moderation workflow can find records still in `unknown` state. Reconciliation can later attach a provider request identifier if one exists.

Do not collapse these states into one `error` column. An explicit `unknown` state tells a reviewer why the item was held and prevents a dashboard from reporting a false success rate.

## How Should Nodejs Handle Moderation API Timeout Errors?

Put the retry loop at the runtime boundary, where the code can see the deadline, response status, attempt number, and tenant metadata together. Business handlers should receive a result object. They should not know whether the third network attempt succeeded.

The loop below uses a generic moderation function. It has no provider-specific route or model identifier, so replacing the remote implementation does not force a rewrite of tenant accounting. The timeout is 800 milliseconds per attempt in this example, and the overall caller budget is separate. Those are policy values to benchmark, not universal defaults.

```ts
type ModerationResponse = {
  decision: "approved" | "rejected";
  requestId?: string;
};

type ModerationClient = (
  input: unknown,
  signal: AbortSignal,
) => Promise<ModerationResponse>;

type ModerationResult =
  | { kind: "decision"; value: ModerationResponse; attempts: number }
  | { kind: "unknown"; attempts: number; reason: "timeout" | "network" }
  | { kind: "failed"; attempts: number; status?: number };

const sleep = (milliseconds: number): Promise<void> =>
  new Promise((resolve) => setTimeout(resolve, milliseconds));

const isRetryableStatus = (status: number): boolean =>
  status === 408 || status === 429 || status >= 500;

export async function moderateWithRetry(
  input: unknown,
  client: ModerationClient,
  options: {
    maxAttempts?: number;
    timeoutMs?: number;
    random?: () => number;
  } = {},
): Promise<ModerationResult> {
  const maxAttempts = options.maxAttempts ?? 3;
  const timeoutMs = options.timeoutMs ?? 800;
  const random = options.random ?? Math.random;

  for (let attempt = 1; attempt <= maxAttempts; attempt += 1) {
    const controller = new AbortController();
    const timer = setTimeout(() => controller.abort(), timeoutMs);

    try {
      const value = await client(input, controller.signal);
      return { kind: "decision", value, attempts: attempt };
    } catch (error: unknown) {
      const timedOut = controller.signal.aborted;
      const status =
        typeof error === "object" && error !== null && "status" in error &&
        typeof error.status === "number"
          ? error.status
          : undefined;
      const retryable = timedOut || status === undefined || isRetryableStatus(status);

      if (!retryable || attempt === maxAttempts) {
        if (timedOut) {
          return { kind: "unknown", attempts: attempt, reason: "timeout" };
        }
        return { kind: "failed", attempts: attempt, status };
      }

      const baseDelay = 100 * 2 ** (attempt - 1);
      const jitter = Math.floor(random() * baseDelay);
      await sleep(baseDelay + jitter);
    } finally {
      clearTimeout(timer);
    }
  }

  throw new Error("unreachable");
}
```

There are two details here that are easy to miss. A fresh `AbortController` is created for each attempt, so an expired signal cannot poison the next one. Also, the final timeout is returned as `unknown`, not `failed`. A network exception without an HTTP status is treated as retryable until the attempt budget is exhausted; that policy should be narrowed if the client can distinguish DNS, connection, and application errors reliably.

The ledger write belongs around this function, not inside it. Write `started` with the tenant ID and request ID before each call. Write `approved` or `rejected` only after a decision arrives. Write `unknown` when the function returns that state. This makes retries visible without making the retry helper responsible for storage.

## How Do You Keep Tenant Cost Visibility Honest?

Per-tenant visibility needs two counters, not one: attempts started and decisions completed. A single “moderation requests” number hides the difference between a tenant with slow upstream responses and a tenant that submitted more content.

For each tenant, chart at least these dimensions over the same time window:

- started attempts, grouped by outcome;
- timeout rate and retry count;
- unknown items waiting for reconciliation;
- latency at the first attempt and at the final decision;
- request size or token-count proxy, if the system already measures it.

The last item is deliberately a proxy. Do not invent a cost formula from a token estimate when the remote billing contract is unknown. A useful internal report says, “Tenant A started 1,240 attempts, 38 ended unknown, and 21 were later reconciled.” It does not claim an exact external charge from that record alone.

This is also where the biggest trade-off appears. Retrying can improve the user-visible completion rate while increasing attempts per item. Stopping immediately can preserve a strict latency budget while pushing work into a queue. I would expose both numbers to the operator and let the tenant policy choose the boundary. Hiding retries produces a neat dashboard and a bad invoice conversation.

## What Should the Tests Prove?

Do not test this with only a successful mock. The dangerous cases are state transitions, and they fit in a small deterministic test matrix:

1. A decision on the first attempt produces one `started` record and one terminal decision.
2. A timeout followed by a decision produces two `started` records and one terminal decision.
3. Three timeouts produce three `started` records and one `unknown` result.
4. A non-retryable client error stops immediately.
5. A late promise from an aborted attempt cannot mutate the next attempt's ledger record.
6. Two tenants retrying at once retain separate request and accounting identities.

Inject `random` as the example does. Inject the clock too if the surrounding code makes timestamps or backoff observable. The exact delay is less important than proving that attempts stop at the configured limit and that jitter prevents every worker from waking on the same millisecond.

Run a small load test with the real payload shapes from the knowledge base. Measure p50 and p95 latency, timeout rate, attempts per decision, and unknown backlog per tenant. A retry policy that looks reasonable in a unit test can overload a shared queue when a school imports a large batch of documents.

## When Is the Runner-Up Better?

The synchronous retry choice is a poor fit when a moderation decision gates a high-stakes action, such as releasing an exam or publishing a student-visible explanation. In that case, fail closed and queue the content for review. The extra waiting is visible, and the policy is easier to explain to teachers.

It is also a poor fit when the caller has a hard sub-second budget and no room for a second attempt. Use one attempt, return a pending state, and let a worker finish the decision out of band. That design gives up immediate certainty to protect the interaction budget.

The right boundary is therefore a product policy encoded as runtime data: tenant, content risk, caller deadline, maximum attempts, and review route. Keep those fields small. If configuring a retry requires a page of nested options, the abstraction has already started charging the developer in glue code.

The practical rule is straightforward: retry transient uncertainty, preserve uncertainty in the data model, and measure attempts per tenant. That gives the application a useful response without claiming a moderation result it never received.

## Sources

References:

- https://developer.mozilla.org/en-US/docs/Web/API/AbortController
- https://nodejs.org/api/globals.html#class-abortcontroller
- https://www.rfc-editor.org/rfc/rfc9110.html
- https://github.com/openai/whisper
