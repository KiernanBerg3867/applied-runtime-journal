# Node.js Password-Reset Delivery (Debugging Malformed SMS, Email, JSON, and Variables)

Short answer: validate a password-reset event once, before choosing email or SMS, then render each channel from the same checked variables and reject the whole dispatch when any recipient or template input is invalid.

Delivery reliability starts before a provider call. A short-lived reset message can be accepted by a queue, rejected later by a channel adapter, or delivered after its token has become useless. I care less about how quickly a demo sends one message than whether the contract makes those states impossible to confuse.

Measure it.

## Why validate before channel selection?

The uncomfortable constraint is expiry. Every retry, queue hop, and second round of parsing consumes part of the useful window, so malformed data should fail while the caller can still replace it. Sending email and SMS validation down separate branches looks tidy at first, but it lets their rules drift: one branch may require `resetUrl`, while the other quietly renders an empty placeholder.

Use one event envelope with one authoritative expiration instant. The channel-specific recipient belongs beside it, but neither adapter gets unvalidated input. This also creates a clean operational distinction: contract rejection means the producer supplied unusable data; delivery failure means a previously valid dispatch did not reach its destination. Don't blend those counters. The smallest useful contract checks four things: the event is the expected kind, the expiry is still in the future, the selected channel has a plausible recipient, and every placeholder required by the selected template exists as a non-empty string. A regular expression can catch obvious shape errors, but it cannot prove that an inbox exists or that a phone can receive a message. I'm not sure any static check can prove reachability; only a delivery result or a separate verification flow can resolve that. This is where a malformed payload earns a precise rejection instead of spending its short lifetime bouncing between adapters.

Reject early.

## How should Node.js debug malformed event notification email SMS payloads and template variables?

Start with an error that points to a field, not a provider response copied into a log. The following TypeScript is deliberately dependency-free so the contract is visible. In a real service, the same shape can live in a JSON Schema validator; the important part is that parsing returns either trusted data or a bounded list of issues. It never returns a half-valid event.

```ts
type Channel = "email" | "sms";

type ResetEvent = {
  event: "password_reset";
  channel: Channel;
  recipient: string;
  expiresAt: string;
  templateVariables: {
    resetUrl: string;
    accountName: string;
  };
};

type ParseResult =
  | { ok: true; value: ResetEvent }
  | { ok: false; issues: string[] };

const emailShape = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
const phoneShape = /^\+[1-9]\d{7,14}$/;

function parseResetEvent(input: unknown, now: Date): ParseResult {
  const issues: string[] = [];
  const value = input as Partial<ResetEvent> | null;

  if (!value || typeof value !== "object") {
    return { ok: false, issues: ["payload must be an object"] };
  }

  if (value.event !== "password_reset") issues.push("event is invalid");
  if (value.channel !== "email" && value.channel !== "sms") {
    issues.push("channel is invalid");
  }

  if (typeof value.recipient !== "string") {
    issues.push("recipient is required");
  } else if (value.channel === "email" && !emailShape.test(value.recipient)) {
    issues.push("recipient is not an email-shaped value");
  } else if (value.channel === "sms" && !phoneShape.test(value.recipient)) {
    issues.push("recipient is not an international phone-shaped value");
  }

  const expiry =
    typeof value.expiresAt === "string" ? Date.parse(value.expiresAt) : Number.NaN;
  if (!Number.isFinite(expiry) || expiry <= now.getTime()) {
    issues.push("expiresAt must be a future timestamp");
  }

  const variables = value.templateVariables;
  if (!variables || typeof variables !== "object") {
    issues.push("templateVariables is required");
  } else {
    if (typeof variables.resetUrl !== "string" || !variables.resetUrl.trim()) {
      issues.push("templateVariables.resetUrl is required");
    }
    if (typeof variables.accountName !== "string" || !variables.accountName.trim()) {
      issues.push("templateVariables.accountName is required");
    }
  }

  return issues.length
    ? { ok: false, issues }
    : { ok: true, value: value as ResetEvent };
}
```

One caveat matters here: `Date.parse` accepts more input than many teams intend. If the producer contract promises a narrower timestamp syntax, enforce that syntax at the boundary as well. The parser above demonstrates the control flow, not a universal timestamp policy.

Then test broken specimens directly. A fixture with an expired timestamp, a fixture missing `resetUrl`, and one invalid recipient per channel will tell you more than a mock that always returns success. Keep secrets out of fixtures; reset URLs can use reserved example domains and fake tokens.

## The dispatch path stays boring

After validation, rendering should be deterministic and transport selection should be exhaustive. No adapter should reinterpret the raw JSON. That removes glue, and it makes the time-to-first-call benchmark honest because setup work is counted rather than hidden in two bespoke mappers.

```ts
type RenderedMessage = {
  destination: string;
  subject?: string;
  body: string;
  expiresAt: string;
};

type Transport = {
  send(message: RenderedMessage): Promise<{ messageId: string }>;
};

function render(event: ResetEvent): RenderedMessage {
  const action = `Reset ${event.templateVariables.accountName}: ${event.templateVariables.resetUrl}`;

  return event.channel === "email"
    ? {
        destination: event.recipient,
        subject: "Reset your password",
        body: action,
        expiresAt: event.expiresAt,
      }
    : {
        destination: event.recipient,
        body: action,
        expiresAt: event.expiresAt,
      };
}

async function dispatch(
  raw: unknown,
  now: Date,
  transports: Record<Channel, Transport>,
): Promise<{ messageId: string }> {
  const parsed = parseResetEvent(raw, now);
  if (!parsed.ok) throw new Error(`invalid reset event: ${parsed.issues.join(", ")}`);

  return transports[parsed.value.channel].send(render(parsed.value));
}
```

Short code is a feature here. The transport interface has one call, the renderer has no I/O, and the parser owns every rejection. I would log a stable event identifier, channel, validation issue code, queue age, expiry time, and transport message identifier; I would not log the reset URL, token, message body, email address, or phone number. NIST's authentication guidance treats secrets and authenticator handling as security-sensitive, which is enough reason to keep raw reset material out of routine telemetry.

Email also has delivery controls outside this function. Google's sender guidance covers authentication, DNS, message formatting, and sending practices. Payload validation cannot substitute for those controls. It only prevents your own malformed event from reaching the delivery layer.

## What changes when this path scales?

At higher volume, I would preserve the boundary and replace its plumbing: compile the JSON Schema once at process start, version the event contract, move valid dispatches through a queue, and stop retries when the remaining lifetime cannot cover the next attempt. The expiry decision should use the worker's current time, not only the producer's original check. Otherwise a valid event can wait in a backlog and emerge already stale.

Observability should separate validation rejection, rendering rejection, transport acceptance, delivery feedback, and expiry cancellation. Those are different events with different owners. A single `send_failed` metric saves configuration up front and charges it back during an incident.

| State | Meaning | Owning boundary |
| --- | --- | --- |
| Validation rejection | Event data cannot produce a message | Producer contract |
| Rendering rejection | Checked variables cannot fill the chosen template | Template build |
| Transport acceptance | Delivery system accepted the message | Channel adapter |
| Delivery feedback | Destination outcome is available | Feedback consumer |
| Expiry cancellation | Useful reset window has closed | Queue worker |

The catch is that dual-channel delivery increases operational surface. It is not suitable when the product cannot maintain separate recipient verification, consent, suppression, and delivery feedback paths. Stick with email when users have verified email addresses and SMS adds no recovery value; stick with SMS only when phone ownership and regional messaging obligations are already handled. A queue is also the wrong default at very small scale if its delay and retry policy are less visible than a direct, bounded call. Your mileage may vary, but the decision should come from measured queue age and delivery outcomes, not an architecture diagram.

I would benchmark three numbers in a staging flow: validation latency, queue age at dispatch, and time from transport acceptance to delivery feedback. No invented target helps. Capture a baseline, set the alert threshold below the reset expiry budget, and rerun the same malformed fixtures on every contract change.

No vanity benchmark.

## References

- https://support.google.com/a/answer/81126
- https://pages.nist.gov/800-63-3/sp800-63b.html
