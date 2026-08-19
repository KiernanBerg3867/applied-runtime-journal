# Password Reset Email Timeouts: Idempotent Retries Without Duplicate Sends

**Short answer:** A timed-out password reset email request is an unknown delivery result, not an automatic resend; duplicate prevention depends on a durable outbox, one stable idempotency key per reset intent, and retries that distinguish a known rejection from an ambiguous timeout.

That distinction matters in a healthtech portal. A patient may submit a contact form because they cannot sign in; the form routes to the access-support queue, and the reset workflow sends an email. If the serverless function gives up before the email API answers, the provider may still have accepted the message. Blindly trying again can put two valid-looking reset messages in a patient's inbox. Don't make the network response your source of truth.

The target is not mythical exactly-once delivery. It is one durable reset intent, at-least-once processing, and controlled side effects.

## How should a serverless API retry password reset email timeouts without duplicate sends?

An API request has three relevant outcomes: accepted, rejected, or unknown. Accepted and rejected are ordinary states. A client-side timeout is different because it says only that the caller stopped waiting. It does not prove that the remote system abandoned the request.

Unknown means stop.

Treating `TimeoutError` as a rejection collapses two states into one. The next invocation then creates a fresh token, generates a fresh message, and calls the email service again. In a serverless runtime, concurrent invocations make that mistake easier: the browser can repeat the form submission, the queue can redeliver work, and a function can be replaced after doing the remote side effect but before recording success. None of those events should mint a second reset intent.

For the contact-form router, I would derive the intent at the boundary. Normalize the account identifier, classify the request into the access-support queue, create the password-reset record once, and commit an outbox row in the same database transaction. The HTTP handler can then return without holding the patient's request open while an email vendor responds. This adds a worker and a table, which is real operational weight, but it removes a remote network call from the form's critical path and gives the team something durable to inspect.

The key must describe the business action, not an invocation. A random value generated inside every retry is useless. A better key is tied to the stored reset intent, such as `password-reset:<intent-id>`, and remains unchanged across function retries, queue redelivery, and process restarts. Store only a hash of the reset token, give the intent an expiry, and avoid logging the email address, raw token, or reset URL. Health data is not required for this workflow, so don't pull it into telemetry.

## The smallest delivery ledger I would ship

Use two layers of deduplication. The database prevents two workers from owning the same outbox item at once, while the email adapter passes the same idempotency key on every attempt when the selected provider supports that contract. The second layer covers the ugly gap where the provider accepts a message and the worker loses the response before it commits `accepted` locally.

If the provider does not offer an idempotent send operation or a reliable lookup by your message key, an ambiguous timeout cannot be made safe by optimistic code. Move the row to `needs_reconciliation` and wait for delivery evidence, query the provider by a stable metadata value if that capability exists, or require an operator-approved resend after a defined interval. I'm not sure there is a universal retry delay that fits every sender; queue age, provider guidance, and the reset token's expiry should decide it. What is universal is the state transition: unknown is not rejected.

Use exponential backoff with jitter for known transient rejections and transport failures that occurred before a request was sent. Cap attempts and total age. Permanent address or policy rejections should stop immediately, while rate-limit responses should honor an explicit retry delay when one is returned. Those rules belong in one worker, not copied into every route and function.

The smallest implementation I would ship looks like this. The storage methods are deliberately abstract: their transaction and lease operations must be atomic in the database, and the email adapter must document whether `idempotencyKey` is actually enforced or merely attached as metadata.

```ts
type DeliveryState =
  | "pending"
  | "sending"
  | "accepted"
  | "needs_reconciliation"
  | "rejected";

type ResetEmail = {
  intentId: string;
  recipient: string;
  resetUrl: string;
  state: DeliveryState;
  attempts: number;
};

interface Outbox {
  claim(intentId: string, leaseMs: number): Promise<ResetEmail | null>;
  markAccepted(intentId: string, providerMessageId: string): Promise<void>;
  markForRetry(intentId: string, nextAttemptAt: Date): Promise<void>;
  markForReconciliation(intentId: string): Promise<void>;
  markRejected(intentId: string, reason: string): Promise<void>;
}

type SendResult =
  | { kind: "accepted"; messageId: string }
  | { kind: "retryable"; retryAfterMs?: number }
  | { kind: "rejected"; reason: string }
  | { kind: "unknown" };

interface EmailAdapter {
  sendPasswordReset(input: {
    to: string;
    resetUrl: string;
    idempotencyKey: string;
  }): Promise<SendResult>;
}

const retryDelay = (attempt: number): number => {
  const ceilingMs = Math.min(60_000, 1_000 * 2 ** attempt);
  return Math.floor(Math.random() * ceilingMs);
};

async function deliverResetEmail(
  intentId: string,
  outbox: Outbox,
  email: EmailAdapter,
): Promise<void> {
  const item = await outbox.claim(intentId, 30_000);
  if (!item || item.state === "accepted" || item.state === "rejected") return;

  const result = await email.sendPasswordReset({
    to: item.recipient,
    resetUrl: item.resetUrl,
    idempotencyKey: `password-reset:${item.intentId}`,
  });

  if (result.kind === "accepted") {
    await outbox.markAccepted(item.intentId, result.messageId);
    return;
  }

  if (result.kind === "unknown") {
    await outbox.markForReconciliation(item.intentId);
    return;
  }

  if (result.kind === "rejected") {
    await outbox.markRejected(item.intentId, result.reason);
    return;
  }

  const delayMs = result.retryAfterMs ?? retryDelay(item.attempts);
  await outbox.markForRetry(item.intentId, new Date(Date.now() + delayMs));
}
```

The adapter should translate its transport's errors into these four results. Keep that translation beside the vendor integration. Otherwise a generic queue consumer ends up guessing whether a status, exception, or closed connection means "try again" — exactly the kind of glue that becomes configuration bloat.

There is another race to close before deployment. Creating the reset intent and inserting its outbox record must share a transaction, with a unique constraint on the form submission or reset-intent identifier. A worker lease must use a conditional update, not a read followed by a write. Those two database guarantees do more for duplicate prevention than another retry library.

## Breaking the worker before users do

Benchmark the workflow as a state machine, not as a single API latency chart. Track counts and age for `pending`, `sending`, and `needs_reconciliation`; acceptance latency from outbox creation; attempts per intent; duplicate form submissions rejected by the unique constraint; and the ratio of ambiguous outcomes. Alert on old work and state transitions that stop moving. Do not put recipient addresses or reset links in metric labels. The useful test is forced ambiguity: make a fake adapter record an acceptance and then return `unknown`, run the worker again, and verify that the same idempotency key is used and the outbox does not create a second intent. Also test two workers claiming one row, a function ending after the send but before the database update, an expired token, a permanent rejection, and a retry delayed beyond the token lifetime. A happy-path mock that answers immediately proves almost nothing. I would also record the provider message identifier after acceptance and propagate the internal intent ID as non-sensitive metadata where supported. That makes reconciliation possible without searching by patient address. Logging should answer "what state is this intent in?" while revealing as little about the person as possible.

Short-lived reset tokens create a product trade-off. Aggressive retry can deliver a link just as it expires; a very long expiry enlarges the window in which a stolen link is useful. Set those values from the portal's security policy, then make the worker stop retrying before the token becomes invalid. The support queue needs a clear terminal state so an agent can issue a new intent instead of reviving an old one.

## The operating boundary at higher scale

At higher volume, partition the outbox by a stable, non-sensitive key; enforce per-domain and provider concurrency limits; and separate reset mail from lower-priority notifications. Backpressure should delay marketing or informational traffic before it delays account access. Keep the message template version on the outbox row so a retry renders the same content rather than silently picking up a mid-flight deployment.

The catch is that an outbox is not suitable when the system has no durable database or queue ownership. For a tiny internal tool, a synchronous send with the submit button disabled may be an acceptable risk, but it cannot promise duplicate prevention after an ambiguous timeout. Stick with synchronous delivery only when duplicate messages have low impact and the team accepts manual investigation. For a healthtech access flow, I would pay the operational cost of durable state.

An SMS fallback does not erase the model. It adds channel-specific limits: SMS encoding affects character count and segmentation, so a fallback template must be tested as SMS rather than treated as a shortened email. It also needs its own consent, routing, idempotency, and delivery states. Adding a second channel before the first state machine is observable usually doubles uncertainty rather than reliability.

The final decision rule is plain. Retry automatically only when the outcome is known to be retryable, preserve one business idempotency key across every attempt, and quarantine ambiguous results unless the downstream system can deduplicate or reconcile them. Fast failure feels clean. Correct uncertainty is better.

## References

- Amazon Simple Email Service documentation: https://docs.aws.amazon.com/ses/latest/dg/Welcome.html
- Twilio, SMS character limits and segmentation: https://www.twilio.com/docs/glossary/what-sms-character-limit
