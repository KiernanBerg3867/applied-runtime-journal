# SMS Batch Alerts: Node.js Status Polling Explained Through 3 Evidence Checks

| Choice | Evidence the application must retain | Main trade-off |
| --- | --- | --- |
| Poll status and keep a local suppression ledger | Observed status, observation time, suppression reason, and send decision | Works with a no-webhook requirement; observations can lag |
| Receive status callbacks and keep the same ledger | Received event, receipt time, suppression reason, and send decision | Better fit when callbacks are permitted; adds an inbound integration |

Polling trades observation freshness for a smaller inbound integration surface. For a fintech web app that requires batch SMS alerts without webhooks, choose polling with a local suppression ledger and a final recipient check before dispatch. Short answer: compare services by the evidence their status reads let you retain and reconcile. A successful API call alone cannot explain why a later alert was allowed.

This note proposes 3 checks: identify the recipient and channel, interpret the observed failure, and record the dispatch decision. The scope is US/EU notification batches, with invalid recipients suppressed before later batches. These are engineering controls for producing evidence. Whether a particular message may legally be sent requires a separate determination for its purpose and jurisdiction.

## How should a web app record SMS batch notification failures?

Start with a concrete question from a reviewer: why did this recipient receive the second balance alert after the first attempt failed? A status screenshot answers only part of that question. The useful record connects the first attempt, the failure classification, the suppression transition, and the second batch's eligibility decision. Keep those links explicit. Preserve the distinction between an event time reported by a messaging service and the time your worker observed it. A polling worker can discover a failure after a later dispatch has already started. Its ledger should reveal that sequence, including the gap in knowledge. Otherwise an apparently complete history can imply that the sender knew something it hadn't yet read. For the illustrative balance alerts below, a reviewer should be able to inspect both the observation that blocked recipient 28 and the decision that excluded that recipient from a later batch, without reconstructing the sequence from unrelated worker logs.

Time matters here.

For the first decision criterion, ask whether a candidate integration can provide a stable attempt identifier and status reads that can be associated with it. Then inspect what the documented statuses mean, how long they remain readable, and whether raw failure details can be retained under your data policy. Treat unanswered questions as acceptance-test requirements. Don't translate every unsuccessful attempt into an invalid recipient.

The email references illustrate why channel boundaries deserve their own check. The linked email API documentation describes an email integration, while the FTC guide addresses commercial email obligations. Neither establishes SMS coverage, SMS status semantics, or permission to send a US/EU text. Email bounce handling can inform the shape of an evidence ledger; its classifications must not silently become phone-number classifications.

## Keep failure classification separate from permission

The second criterion is control over the suppression decision. Give invalid-address evidence, a recipient's messaging preference, and an unresolved delivery result distinct representations. The proposed rule here is conservative: an explicit invalid-recipient classification blocks subsequent attempts for that recipient and channel, while an unknown result blocks automatic redispatch until reconciled. Neither state grants permission to send.

That trade-off can delay a legitimate alert.

It also makes the uncertainty inspectable. Decide which message classes may wait and what separate escalation path a delayed financial notification needs before choosing the default. A generic retry switch is too blunt for that decision.

Use channel-scoped recipient keys. In the example below, `sms:recipient-17` and `email:recipient-17` would be different keys; an email failure would not, by itself, disable that person's SMS alerts. Keep actual contact details out of routine worker logs when an internal key is sufficient. The ledger still needs a controlled way to resolve the key during an authorized review.

Reactivation deserves a recorded reason and a new eligibility decision. Don't let a subsequent ambiguous status erase an existing invalid-recipient suppression. A change to the recipient's contact information should also have an explicit lifecycle, rather than inheriting an old number's history without explanation. These are proposed application rules, not claims about a particular service's behavior or a legal retention requirement.

## A minimal polling boundary in TypeScript

Keep the glue small. Normalize each service's documented results behind one interface, and let the application own the policy. The following code is a proposed Node.js worker boundary. Its identifiers and statuses are illustrative application data, not an external API contract.

```ts
type Result = "pending" | "delivered" | "invalid" | "unknown";
type Attempt = { id: string; recipientKey: string };
type Observation = {
  attemptId: string;
  recipientKey: string;
  observedAt: string;
  result: Result;
};
interface StatusReader {
  read(attemptId: string): Promise<Result>;
}
interface Ledger {
  // Atomically append evidence and apply any invalid-recipient block.
  // Repeated observations must not remove an existing block.
  record(observation: Observation): Promise<void>;
}
async function reconcile(
  attempts: readonly Attempt[],
  reader: StatusReader,
  ledger: Ledger,
  now: () => Date,
): Promise<void> {
  for (const attempt of attempts) {
    let result: Result;
    try {
      result = await reader.read(attempt.id);
    } catch {
      result = "unknown";
    }
    await ledger.record({
      attemptId: attempt.id,
      recipientKey: attempt.recipientKey,
      observedAt: now().toISOString(),
      result,
    });
  }
}
const exampleBatch: readonly Attempt[] = [
  { id: "balance-alert-a", recipientKey: "sms:recipient-17" },
  { id: "balance-alert-b", recipientKey: "sms:recipient-28" },
  { id: "balance-alert-c", recipientKey: "sms:recipient-39" },
];
```

The interface deliberately makes evidence persistence mandatory. A status read failure produces an unknown observation; a ledger write failure propagates, so the caller cannot treat the reconciliation pass as complete. The adapter must preserve diagnostic details elsewhere in the restricted evidence store if reviewers need to distinguish transport failures from unrecognized results. Collapsing them in the policy interface shouldn't destroy the original evidence.

This is one boundary, not a complete sender. The dispatch path must check current suppression and permission state before each attempt and record its decision. Materializing an eligible batch hours earlier would leave a stale snapshot. There is also a race between that final check and external submission: define the dispatch claim and suppression ordering in the durable store, and retain enough timestamps to explain an attempt already in flight when new evidence arrives.

Keep that race visible.

## Benchmark the evidence path before deployment

Use the 3 illustrative recipients as a deterministic fixture: return delivered for the first, invalid for the second, and throw a read error for the third. The expected evidence is one observation per attempt, a block for the second recipient, and unresolved state for the third. Run the next dispatch decision against those results. Assert that an existing block survives a later unknown observation.

Measure rather than guess.

Capture time to the first persisted observation, elapsed time from an available invalid result to a committed suppression, and the number of status reads per reconciled batch. These are proposed benchmark measurements; no performance results are claimed here. Compare the same fixture, polling schedule, and concurrency limit across adapters. A fast initial request is a weak result if the subsequent evidence path requires undocumented manual work.

Also test a worker restart after a successful read but before persistence, overlapping polling workers, and a ledger outage. Check what happens when a service no longer retains an old attempt. That last case should produce an explicit evidence gap and an escalation decision, rather than an invented terminal status. Deployment should include a reconciliation backlog signal and an alert for repeated persistence failures.

Avoid a configuration forest. Start with explicit controls for polling interval, concurrency, and unresolved-result handling. Tune them against measured lag and documented rate limits. Request volume, evidence storage, and operator review all belong in the cost model, but a unit-price comparison cannot establish whether the history is defensible.

## When callbacks are the better runner-up

Choose callbacks when an inbound receiver is acceptable and the documented event contract better fits the application's freshness target. They still need durable evidence capture, duplicate handling, and a dispatch gate. Polling remains useful as a reconciliation method if the integration supports the required reads; verify that support instead of assuming it.

For the stated no-webhook requirement, the decision remains polling with retained observations and explicit suppression decisions. Approve the integration only after the fixture can explain why the invalid recipient was blocked, why the unresolved attempt waited, and what the sender knew when it dispatched the next batch. That's a useful acceptance test for a fintech notification system.

## References

These sources are scoped to email context. They don't substantiate SMS availability, US/EU SMS consent rules, retention periods, or status guarantees. The polling interfaces, fixture, and acceptance criteria above are proposed application design.

## Sources

- Email API documentation, for the channel boundary discussed above: https://resend.com/docs/introduction
- FTC, CAN-SPAM Act compliance guide for business, for commercial email scope: https://www.ftc.gov/business-guidance/resources/can-spam-act-compliance-guide-business
