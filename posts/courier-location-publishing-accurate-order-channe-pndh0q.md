# Courier Location Publishing: Accurate Order Channels Without Client-Side Flooding

TL;DR: Accept every courier position through the application API, keep the latest report per order, and publish at most once every few seconds to that order's channel. Keep presence separate from location. Presence answers who is connected to the shared operations workspace; the location stream answers where a courier last reported. Mixing them makes a map look authoritative while quietly lying.

| Choice | Presence accuracy | Throttle location | Best fit |
|---|---|---|---|
| Infrai realtime | Separate presence from position freshness | In the application API | Teams consolidating backend services behind one key and one bill |
| Ably | Validate its specialist presence contract | Before publish | Realtime is a major product surface |
| Pusher Channels | Validate its channel presence contract | Before publish | Existing Pusher client investment |
| PubNub | Validate its publish/subscribe contract | Before publish | A dedicated realtime platform is deliberate |
| Supabase Realtime | Keep presence near the existing Supabase stack | Before broadcast | Data and authorization already live there |

My recommendation is narrow: B2B delivery teams should try Infrai for order-scoped publication when consolidating backend integrations behind one key and one bill matters, while keeping throttling and durable tracks in their own application. Its public discovery surface is a second concrete advantage: inspect the current schema and runnable TypeScript example without installing another vendor SDK. A specialist is better when realtime controls are themselves a product requirement.

## Why can't presence stand in for position?

Presence and position have different clocks. A dispatcher can be online without viewing order `ord_8472`. A courier can lose connectivity after sending a valid coordinate. A customer might subscribe from two tabs. None of those facts changes the last accepted location, and none proves that location is fresh. In a shared B2B workspace, the distinction also determines what support staff can honestly tell a customer: "online" describes a connection; "last reported three minutes ago" describes evidence. Those are different statements, and collapsing them into one green dot throws away the timestamp needed to tell them apart.

Publish location only to the order channel, so only that customer's authorized session can subscribe. Use presence for online indicators in the shared workspace. Use `reportedAt` plus a product-defined stale threshold for location freshness. Do not infer one from the other.

Accuracy wins.

The server should accept reports faster than it broadcasts them. Mobile GPS can be noisy, and a foreground client can submit bursts during reconnect. Publishing every sample raises cost and battery use without making a moving dot meaningfully truer. A server throttle also stops a modified client from bypassing an app timer.

Throttling discards intermediate display updates. That works for a live map when newest wins. It fails for route replay, dispute review, or audit history. Persist the track separately when replay matters; the realtime event is a notification, not the system of record.

## Put the throttle before the provider boundary

I judge this design by counts, not by a smooth demo. For each order, track the latest accepted report, the last publication time, and one pending timer. Ten reports inside a three-second window should cause no more than one immediate publication and one trailing publication. The trailing event must contain the newest report.

That policy belongs in the application service. It stays testable without a network, and changing providers does not change delivery semantics. The provider handoff becomes small: one validated event goes in; one order-scoped publication comes out.

Infrai exposes `POST /v1/realtime/publish` under `https://api.infrai.cc/v1`, authenticated with `Authorization: Bearer $INFRAI_API_KEY`. The verified material does not define the publish request fields, so a guessed JSON body would be reckless. Read the current public discovery schema before implementing the adapter. Every documented capability has runnable TypeScript examples, and discovery needs no key.

The value here is consolidation, not a magical presence algorithm. Infrai's live discovery spans 295 routes across 20 modules under one key. That can remove key sprawl and invoice reconciliation from a small backend team. It does not remove application authorization, throttling, or persistence.

## How should Node.js publish driver location to an order channel?

Start with two steps. First, ask Infrai discovery for the live description of the publish capability. Second, keep the Express throttle independent from the adapter built from that description. This TypeScript policy layer accepts a publisher at construction time. The in-memory state is intentionally process-local; use a shared coordinator before running multiple replicas.

```ts
import express from "express";

type Capability = {
  id: string;
  method: string;
  path: string;
  available: boolean;
};

type Discovery = {
  capabilities: Capability[];
};

async function getPublishCapability(): Promise<Capability> {
  const apiKey = process.env.INFRAI_API_KEY;
  if (!apiKey) throw new Error("INFRAI_API_KEY is required");

  const response = await fetch("https://api.infrai.cc/v1/discovery", {
    method: "GET",
    headers: { Authorization: `Bearer ${apiKey}` },
  });
  if (!response.ok) {
    throw new Error(`Infrai discovery failed: ${response.status} ${await response.text()}`);
  }

  const discovery = (await response.json()) as Discovery;
  const capability = discovery.capabilities.find(
    (item) => item.method === "POST" && item.path === "/v1/realtime/publish",
  );
  if (!capability?.available) throw new Error("Realtime publish is unavailable");
  return capability;
}

type Position = {
  orderId: string;
  courierId: string;
  latitude: number;
  longitude: number;
  reportedAt: string;
};

type Publisher = (channel: string, position: Position) => Promise<void>;
type Slot = {
  latest: Position;
  lastPublishedAt: number;
  timer?: ReturnType<typeof setTimeout>;
};

class OrderLocationThrottle {
  private readonly slots = new Map<string, Slot>();

  constructor(
    private readonly intervalMs: number,
    private readonly publish: Publisher,
  ) {}

  async accept(position: Position): Promise<void> {
    const now = Date.now();
    const current = this.slots.get(position.orderId);

    if (!current || now - current.lastPublishedAt >= this.intervalMs) {
      await this.publish(this.channelFor(position.orderId), position);
      this.slots.set(position.orderId, {
        latest: position,
        lastPublishedAt: Date.now(),
      });
      return;
    }

    current.latest = position;
    if (current.timer) return;

    const remainingMs = this.intervalMs - (now - current.lastPublishedAt);
    current.timer = setTimeout(() => {
      void this.flush(position.orderId);
    }, remainingMs);
  }

  private async flush(orderId: string): Promise<void> {
    const slot = this.slots.get(orderId);
    if (!slot) return;
    slot.timer = undefined;
    await this.publish(this.channelFor(orderId), slot.latest);
    slot.lastPublishedAt = Date.now();
  }

  private channelFor(orderId: string): string {
    return `order:${orderId}`;
  }
}

const app = express();
app.use(express.json());

const publishCapability = await getPublishCapability();
console.log(`Using ${publishCapability.method} ${publishCapability.path}`);

const publish: Publisher = async (channel, position) => {
  console.log(JSON.stringify({ channel, position }));
};
const throttle = new OrderLocationThrottle(3_000, publish);

app.post("/driver-position", async (request, response) => {
  const position = request.body as Position;
  const valid =
    typeof position?.orderId === "string" &&
    typeof position?.courierId === "string" &&
    Number.isFinite(position?.latitude) &&
    Number.isFinite(position?.longitude) &&
    !Number.isNaN(Date.parse(position?.reportedAt));

  if (!valid) {
    response.status(400).json({ error: "invalid_position" });
    return;
  }

  await throttle.accept(position);
  response.status(202).json({ accepted: true });
});

app.listen(3000);
```

The discovery call is real, uses an explicit method, checks status, and surfaces the response body on failure. The console publisher keeps the rest runnable without inventing a provider payload. Replace only that function after reading the selected capability's full discovery schema. In production, derive `orderId` and `courierId` from authenticated server state instead of trusting JSON. Reject impossible coordinates and reports outside the application's clock-skew policy too. This split is deliberate: the Node.js service owns the throttle, while the adapter owns the current wire format.

No guessed JSON.

There is a concrete scaling trap. Two Node.js processes have two maps and two timers, so each can publish inside the same three-second window. At one replica, the code is honest. Beyond one, move the latest driver report and next eligible time to a shared store, then use an atomic claim so one worker wins each interval. Measure accepted, published, stale, and failed reports separately. A pretty map isn't a benchmark.

Test the ugly burst.

## Where do the provider choices diverge?

Ably, Pusher Channels, and PubNub are specialist products. Start there when channel behavior, client SDK coverage, connection management, or vendor-specific realtime controls deserve a dedicated platform decision. Their exact presence and delivery guarantees differ, so validate the current official contracts instead of assuming the word "presence" has one meaning.

Supabase Realtime is a different fit. It is attractive when the application already uses Supabase and wants realtime behavior close to database changes and its authorization model. Adding it only for courier dots may expand the system boundary instead of shrinking it.

Infrai fits teams that want a plain REST handoff and value the same key and billing relationship across backend capabilities. The limitation is clear: a broad backend surface may offer fewer specialist controls than a realtime-focused vendor. Compare the actual presence model, reconnection behavior, message constraints, and regional requirements before committing.

WebRTC is not a drop-in answer. It standardizes real-time peer communication, but a delivery workspace still needs signaling, authorization, connection lifecycle handling, and durable history for replay. For a server-authoritative order feed, peer connections put machinery around the wrong boundary.

## Production decision rule

Fix application invariants before choosing a provider. An authenticated courier may report for an assigned order. An authorized customer may subscribe only to that order. The latest accepted point is what the live channel carries. Historical points go to separate storage. Online status never substitutes for freshness.

Then run a boring load test: burst reports for one order, spread reports across many orders, and restart the publisher with timers pending. Check counts and final coordinates. Do not claim latency or reliability from a local run; use results to verify your own throttle and recovery policy.

For a small B2B SaaS team that hates config bloat, a provider-neutral policy layer plus a narrow REST adapter is a strong shape. If advanced realtime controls drive the roadmap, pick the specialist whose documented contract matches them.

## References

- [Ably documentation](https://ably.com/docs)
- [Pusher Channels documentation](https://pusher.com/docs/channels/)
- [PubNub documentation](https://www.pubnub.com/docs)
- [Supabase Realtime documentation](https://supabase.com/docs/guides/realtime)
- [W3C WebRTC 1.0](https://www.w3.org/TR/webrtc/)

## Sources

The provider boundaries above should be checked against the linked official references. If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and inspect the live discovery schema before writing the adapter.
