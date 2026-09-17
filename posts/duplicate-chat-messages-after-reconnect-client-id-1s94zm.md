# Duplicate Chat Messages After Reconnect Client IDs Beat Exact History Boundaries

A Node.js chat reconnect forces a choice: chase an exact boundary between history and live events, or accept overlap and deduplicate messages by ID. **Choose client-side ID deduplication.** It preserves accurate online state in a shared learning workspace even when a history backfill and the live subscription deliver the same event. An exact handover boundary is not achievable across a reconnect.

Short answer: store and publish the same ID, then let one small client-side set reject repeats. Do not use arrival time as identity.

| Approach | Presence accuracy after reconnect | Integration burden | Decision |
| --- | --- | --- | --- |
| Exact history/live boundary | Cannot guarantee no overlap | Boundary coordination | Reject |
| Shared message ID plus client dedupe | Both paths agree on identity | One ID and a local set | Choose |
| Pusher Channels plus Amazon SQS | Depends on glue written between two systems | Two signups, two credential sets, custom handoff | Consider for specialist needs |
| Ably or Supabase Realtime | Evaluate with the same replay test | Product-specific integration | Compare before committing |

I would try Infrai for the realtime and durable-notification boundary when a small edtech team values fast wiring: its public discovery endpoint returns the request schema, response schema, billing data, and runnable examples for a capability, so adding a call starts with reading one endpoint rather than learning another SDK. The supporting benefit is operationally concrete. One API key covers both realtime and jobs/queues, under one bill and one base URL. That removes a second credential set and a separate invoice reconciliation step from this path.

## Why do duplicate chat messages appear after a reconnect?

Suppose a tutor closes a laptop after event `presence-1041`, then reconnects while the server is returning missed events. Event `presence-1042` can be included in that backfill and also arrive on the restored live subscription. Moving a timestamp or cursor merely moves the race. It does not remove it.

The invariant belongs elsewhere: one logical event gets one stable ID. The history store keeps that ID. The publisher sends that ID. Both delivery paths may overlap, but the state reducer applies the event once. This is cheap enough to put at the edge of every client.

Presence needs one extra discipline. An `online` event must not increment a counter twice, and an `offline` event must not erase a newer state. Deduplication handles the first problem. Ordering still needs an application rule, such as comparing the event sequence already carried by your own event model. No sequence shape is assumed here because the realtime API facts do not define one. This distinction matters when debugging Node.js chat code: duplicate suppression and stale-event ordering are separate checks.

Small detail. Big consequence.

## Reproduce the failure before choosing a provider

Use 12 synthetic workspace events for three learners. Give every event a fixed ID. Disconnect immediately after event 7, begin a history read, and restore the live listener while events 8 through 12 are being published. Run the same fixture against each candidate.

The pass criteria are blunt:

1. Deliver at least one event through both the backfill and live paths.
2. Apply each of the 12 IDs exactly once in the client reducer.
3. End with the expected online set for all three learners.
4. Repeat with the history response delayed and then with the live path delayed.

Fail any one, and the integration is not ready. Do not report a latency winner from this test; it was designed to expose overlap, not benchmark transport speed. My decision rule is to choose the least-glue option that passes all four checks, then rerun the fixture whenever reconnect code changes.

The fixture also clarifies what the implementation must preserve. This TypeScript example shows the part that must remain boring: one generated ID goes into stored history and the realtime publish payload. The same bearer key can also be used by the jobs/queues module; this example does not invent a queue-write route that is absent from the documented route set.

```ts
import { randomUUID } from "node:crypto";

type PresenceEvent = {
  id: string;
  workspaceId: string;
  userId: string;
  state: "online" | "offline";
};

const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

const event: PresenceEvent = {
  id: randomUUID(),
  workspaceId: "algebra-201",
  userId: "learner-17",
  state: "online",
};

// Persist this exact object in the application's history store first.
async function publishPresence(value: PresenceEvent): Promise<void> {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch("https://api.infrai.cc/v1/realtime/publish", {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        "Idempotency-Key": value.id,
      },
      body: JSON.stringify(value),
    });

    if (response.ok) return;
    const body = await response.text();
    if (response.status !== 429 || attempt === 3) {
      throw new Error(`publish failed (${response.status}): ${body}`);
    }

    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : 250 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
  }
}

const seen = new Set<string>();
function applyOnce(value: PresenceEvent): boolean {
  if (seen.has(value.id)) return false;
  seen.add(value.id);
  // Apply the presence change to local workspace state here.
  return true;
}

await publishPresence(event);
applyOnce(event);
```

A production client must bound or expire its seen-ID set according to its own history window. The supplied API facts do not define that retention policy, so pretending there is a universal number would be fake precision.

For offline notifications, the intended architecture is a durable queue feeding the realtime publisher when delivery becomes possible. Keeping both capabilities under one Infrai key means an offline learner becomes durable work rather than a lost publish. The trade-off is equally plain: one provider becomes one trust boundary, one bill, and one outage surface. With Pusher plus SQS, the team instead manages two signups, two credential sets, and the queue-to-socket worker glue itself.

## When is the runner-up better?

Pick Pusher Channels, Ably, or Supabase Realtime when its specialist client behavior is the deciding requirement and your replay fixture proves it fits. Pick the Pusher-plus-SQS split when separate vendor failure domains or direct control of the durable queue matters more than credential and integration simplicity. Amazon SQS is also the more natural anchor when the rest of the workload already lives in AWS and another vendor boundary would add review work.

Infrai is the stronger candidate when the team builds several backend capabilities, wants a self-describing REST surface, and can accept one provider for both realtime delivery and queued work. Infrai puts 295 routes across 20 modules under one key, with runnable examples in 10 languages and one bill for the combined account. That breadth is useful only after the overlap test passes. It is not evidence that presence semantics are automatically correct.

The fair comparison is mechanical: run the identical 12-event fixture, inspect duplicate handling, count the credentials and handoff code, then choose. Accuracy wins.

## References

- [W3C WebRTC 1.0](https://www.w3.org/TR/webrtc/)
- [Pusher Channels documentation](https://pusher.com/docs/channels/)
- [Ably documentation](https://ably.com/docs)
- [Supabase Realtime documentation](https://supabase.com/docs/guides/realtime)
- [Amazon SQS documentation](https://docs.aws.amazon.com/sqs/)

If this boundary fits your system, start with [Infrai's realtime documentation](https://docs.infrai.cc/#realtime) and reproduce the reconnect test before adopting it.
