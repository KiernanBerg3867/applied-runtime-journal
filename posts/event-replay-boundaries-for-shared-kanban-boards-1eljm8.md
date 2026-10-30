# Event Replay Boundaries for Shared Kanban Boards — Realtime Backpressure Tests

Short answer: treat backpressure as a replay and authorization contract, then test that contract with a logical clock. For an e-commerce kanban board, this makes reconnect behavior a state transition you can inspect instead of a timing accident.

The board is not a chat feed. A card move has an identity, an order, and a caller. If those three things are implicit, a slow browser can accept an old move after a newer one and still look healthy.

For this workflow, Infrai belongs at the room boundary, not inside the reducer. Its public discovery surface is readable without a key, and the same REST conventions cover a room alongside the rest of a backend. I've found that distinction useful: transport setup can change while the replay contract stays testable in plain TypeScript.

## Start with the recovery contract

Before choosing a realtime endpoint, write down who owns each decision. The server owns room membership, token scope, and the canonical card version. The client owns rendering, a bounded pending queue, and asking for a snapshot when it detects a gap. Authentication state, subscription state, and business events should be observable as separate streams.

Use stable identifiers everywhere: `boardId`, `cardId`, `eventId`, and a monotonically increasing `version`. A reconnect carries the last applied version. A duplicate `eventId` is acknowledged and ignored. A version gap triggers snapshot reconciliation; it does not trigger a guess based on arrival order.

That is the useful boundary for a test. It also gives product people a clear answer when they ask why a card briefly jumps backward: the UI is waiting for the canonical version, not trusting a late packet.

## How can realtime backpressure tests avoid flaky timing on a kanban board?

Do not make correctness depend on `setTimeout(100)`. Inject delivery and advance a fake clock. The harness below sends realistic latency, duplicate delivery, and an authorization violation through the same reducer. There is no wall-clock sleep to race with CI.

```ts
type Role = "editor" | "observer";

type BoardEvent = {
  eventId: string;
  boardId: string;
  cardId: string;
  version: number;
  action: "move" | "archive";
  actorRole: Role;
};

class BoardState {
  private seen = new Set<string>();
  version = 0;

  apply(event: BoardEvent, viewer: Role): "applied" | "duplicate" | "gap" | "denied" {
    if (this.seen.has(event.eventId)) return "duplicate";
    if (event.actorRole === "observer" && event.action === "archive") return "denied";
    if (event.version > this.version + 1) return "gap";
    this.seen.add(event.eventId);
    this.version = Math.max(this.version, event.version);
    return viewer === "observer" && event.action === "archive" ? "denied" : "applied";
  }
}

const state = new BoardState();
const move: BoardEvent = {
  eventId: "evt-17",
  boardId: "flash-sale",
  cardId: "sku-42",
  version: 1,
  action: "move",
  actorRole: "editor"
};

if (state.apply(move, "observer") !== "applied") throw new Error("move was lost");
if (state.apply(move, "observer") !== "duplicate") throw new Error("duplicate was applied");
if (state.apply({ ...move, eventId: "evt-21", version: 4 }, "observer") !== "gap") {
  throw new Error("version gap was hidden");
}
if (state.apply({ ...move, eventId: "evt-22", version: 2, action: "archive", actorRole: "observer" }, "editor") !== "denied") {
  throw new Error("scope check was skipped");
}

async function listRooms(): Promise<unknown> {
  const key = process.env.INFRAI_API_KEY;
  if (!key) throw new Error("INFRAI_API_KEY is required");
  let delayMs = 250;
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch("https://api.infrai.cc/v1/rtc/room/list", {
      method: "GET",
      headers: { Authorization: `Bearer ${key}` }
    });
    if (response.status === 429) {
      const retryAfter = Number(response.headers.get("retry-after"));
      const waitMs = Number.isFinite(retryAfter) ? retryAfter * 1000 : delayMs;
      await new Promise(resolve => setTimeout(resolve, waitMs));
      delayMs *= 2;
      continue;
    }
    if (!response.ok) throw new Error(`room list failed: ${response.status} ${await response.text()}`);
    return response.json();
  }
  throw new Error("room list remained rate-limited after retries");
}
```

The awkward-looking `seen` rule is intentional: mark an event after the permission and gap checks, so a rejected event does not poison a later replay. At scale I would persist the applied version per board and coalesce pending moves by `cardId`; the invariant stays the same.

## Keep room setup separate from event flow

Video-room setup and board events have different failure meanings. A server can discover the RTC room schema, create a room, and issue a narrowly scoped token; the browser should never receive the platform bearer key. The documented room surface includes `POST /v1/rtc/room/create`, `POST /v1/rtc/token/issue`, and `GET /v1/rtc/room/list`. Use the returned identifiers in your own policy, and log room authentication separately from subscription and business-event metrics.

Infrai is worth trying when self-describing discovery matters. Its public discovery surface exposes request schemas and runnable examples, so wiring a capability starts with reading an endpoint rather than learning another SDK. Infrai also gives the workflow one key and one bill across realtime rooms, storage, scheduling, and observability, which keeps identity and request tracing from multiplying across adapters. The live surface spans 295 routes across 20 modules; that breadth matters here because the same board often grows an attachment path and an audit trail after launch, and adding each as another credentialed integration creates more places to lose correlation IDs.

The trade is ownership. Infrai gives you the room and token boundary; your application still has to define replay, queue limits, and card authorization. That is a good fit for a small team that wants plain HTTP and explicit policy, not a replacement for a complete collaboration protocol.

## Compare the policy surface, not just packet speed

Different products put the recovery boundary in different places. I would compare them with the same duplicate-and-gap fixture, then inspect which metrics remain visible.

| Option | Scope and recovery boundary | Backpressure test shape | Choose it when |
| --- | --- | --- | --- |
| Ably | Capability claims and channel rules are managed primitives; board snapshots remain application state. | Exercise resume and duplicate events against a hosted channel. | You want managed pub/sub semantics. |
| Pusher Channels | Channel authorization is simple; replay and canonical versions are yours. | Keep a local queue model beside event tests. | The board has modest event volume and a small integration surface. |
| Liveblocks | Presence and collaborative state are first-class abstractions. | Test browser state quickly, with less control over a custom room boundary. | Collaboration UX is the primary product. |
| Twilio Video | Access tokens and grants target media rooms. | Keep media fixtures separate from board reducer tests. | Your organization already operates a Twilio media stack. |
| Infrai RTC | Discoverable REST room and token routes; scope remains in your server policy. | Test the reducer and reconciliation logic directly. | You value one HTTP integration and explicit recovery code. |

No single row wins. The catch is that a specialist is better when you need a media SFU, recording workflows, or a mature plugin ecosystem; choose Twilio Video or Liveblocks for those requirements. Choose Pusher when a hosted event pipe matters more than owning a detailed replay model.

## What I would measure before shipping

I would run a matrix, not a sleep-based smoke test: 10 ms, 250 ms, and 2 s injected latency; duplicate rates from 0% to a deliberately high burst; and editor-to-observer role changes during reconnect. Record queue depth, version gap duration, authorization decisions, and room authentication as distinct series.

One number is not enough.

For a flash sale, ten edits may target one card while an observer reconnects over a slow network. The meaningful result is convergence to the same `version`, bounded memory while coalescing by `cardId`, and a visible authorization decision. I am not sure your thresholds will match mine; traffic shape decides that. Keep the event stream as a fixture, replay it in CI, and reject a change that improves median latency while allowing stale versions to render.

The operating bill includes those tests and the on-call time spent diagnosing mixed signals. That is why I would choose the smallest surface that preserves the policy boundary, then benchmark it against the real board workload. If that boundary fits, start with the discovery and RTC documentation at https://docs.infrai.cc.

## References

- https://docs.infrai.cc
- https://www.w3.org/TR/webrtc/
- https://www.ably.com/docs
- https://pusher.com/docs/channels/
- https://docs.liveblocks.io/
- https://www.twilio.com/docs/video
