# Secure SMS OTP for Automotive Service Updates — Rate Limits, Lockouts, Replay Defense

Short answer: a secure SMS OTP login flow for automotive service updates needs per-user, per-IP, and per-device limits before any message is sent, followed by short expiry, bounded retries, temporary lockout, and single-use verification. The SMS API delivers and checks codes; your SaaS backend owns abuse prevention.

That split matters across US and EU traffic because native geography throttles and country-price kill switches are not provided. Put country policy ahead of delivery. Don't discover a blocked market after messages have already left.

## What should a secure SMS OTP login flow rate-limit and lock out?

Start with three counters, not one. A per-user counter stops repeated sends to the same account. A per-IP counter catches one origin rotating through many phone numbers. A per-device counter covers shared networks where an IP-only rule would either miss abuse or punish an entire dealership. Check all three before calling the provider, and reject when any counter is over its threshold.

The exact thresholds aren't universal. I'm not sure a single set can be defended for both a consumer booking portal and a service-advisor console; traffic data, account value, carrier delivery time, and support capacity should settle them. What is universal is the order: normalize the account and device identifiers, evaluate country policy, check suppression, consume the rate-limit allowance atomically, and only then request an OTP. A concurrent pair of requests must not both pass a read-then-write counter.

Keep verification state server-side: an opaque challenge ID, a code digest rather than the code, an expiry, failed-attempt count, lockout deadline, and a consumed timestamp. Short expiry narrows the guessing window. A maximum attempt count ends it. Temporary lockout slows repeated attacks. Marking a successful challenge consumed blocks replay.

One detail is easy to miss — resend and verify are different operations. A resend must pass the send limits again, while a failed verification consumes an attempt on the existing challenge. Otherwise an attacker gets a fresh delivery path or an unlimited guessing loop by switching endpoints.

No shortcuts.

## The smallest state machine that keeps provider calls behind policy

The useful implementation boundary is a gate that returns a decision before the provider adapter runs. This TypeScript sketch contains the business-layer logic: it rejects disallowed countries, respects suppression, atomically increments scoped counters, locks a challenge after repeated failures, expires it, and consumes it exactly once. The concrete values are policy inputs because the evidence doesn't establish safe universal numbers.

```ts
import { createHash, timingSafeEqual } from "node:crypto";

type Challenge = {
  digest: Buffer;
  expiresAt: number;
  attempts: number;
  maxAttempts: number;
  lockedUntil: number;
  consumedAt?: number;
};

type SendInput = { userId: string; ip: string; deviceId: string; country: string };

type Dependencies = {
  countryAllowed(country: string): boolean;
  isSuppressed(userId: string): Promise<boolean>;
  consumeLimit(scope: string): Promise<boolean>;
};

export async function maySend(input: SendInput, deps: Dependencies): Promise<boolean> {
  if (!deps.countryAllowed(input.country) || await deps.isSuppressed(input.userId)) return false;

  const scopes = [
    `user:${input.userId}`,
    `ip:${input.ip}`,
    `device:${input.deviceId}`,
  ];
  const decisions = await Promise.all(scopes.map((scope) => deps.consumeLimit(scope)));
  return decisions.every(Boolean);
}

const digest = (challengeId: string, code: string): Buffer =>
  createHash("sha256").update(`${challengeId}:${code}`).digest();

export function verifyOnce(
  challengeId: string,
  code: string,
  challenge: Challenge,
  now: number,
  lockoutMs: number,
): boolean {
  if (challenge.consumedAt !== undefined || now >= challenge.expiresAt) return false;
  if (now < challenge.lockedUntil || challenge.attempts >= challenge.maxAttempts) return false;

  challenge.attempts += 1;
  const candidate = digest(challengeId, code);
  const matches = candidate.length === challenge.digest.length &&
    timingSafeEqual(candidate, challenge.digest);

  if (!matches) {
    if (challenge.attempts >= challenge.maxAttempts) challenge.lockedUntil = now + lockoutMs;
    return false;
  }

  challenge.consumedAt = now;
  return true;
}
```

In production, the counters and challenge mutation need an atomic datastore operation, not process memory. Bind the challenge to the intended account and login transaction as well.

The adapter below is intentionally schema-neutral: the discovery document supplies the request JSON Schema, while this function supplies the HTTP behavior. Passing an `unknown` body avoids teaching fields that haven't been verified here. Validate the body against discovery before calling it. The hosted delivery route is `POST /v1/sms/otp`, while verification uses `POST /v1/sms/verify`; every request sets its method, authenticates from the environment, handles 429 with `Retry-After` or exponential delay, and surfaces a real 4xx response body.

```ts
const wait = (ms: number): Promise<void> =>
  new Promise((resolve) => setTimeout(resolve, ms));

export async function sendOtp(body: unknown): Promise<unknown> {
  const apiKey = process.env.INFRAI_API_KEY;
  const baseUrl = process.env.INFRAI_BASE_URL;
  if (!apiKey) throw new Error("INFRAI_API_KEY is required");
  if (!baseUrl) throw new Error("INFRAI_BASE_URL is required");

  const idempotencyKey = crypto.randomUUID();

  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(`${baseUrl}/v1/sms/otp`, {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        "Idempotency-Key": idempotencyKey,
      },
      body: JSON.stringify(body),
    });

    if (response.status === 429 && attempt < 3) {
      const retryAfter = Number(response.headers.get("Retry-After"));
      await wait(Number.isFinite(retryAfter) ? retryAfter * 1_000 : 250 * 2 ** attempt);
      continue;
    }
    if (!response.ok) throw new Error(`OTP request failed (${response.status}): ${await response.text()}`);
    return response.json();
  }

  throw new Error("OTP request exhausted its retry budget");
}
```

Generate one idempotency key for a logical write and retain it across transport retries, as the adapter does. More important, rerun the business checks before a send retry. A retry that outlives the original allowance or challenge should stop.

Infrai's plain REST surface is attractive here because there is no SDK or client-library version to carry, and one key can cover the communication workflow. The application gate still decides whether either call is allowed.

## Country rules and suppression belong before delivery

For a US/EU automotive SaaS, make the allowed-country set explicit in backend configuration and derive the country from a normalized destination, not a client-supplied dropdown. A deny decision should happen before an OTP request. Geography anti-fraud controls are not native, so leaving this to the delivery layer creates a policy hole; a price-based country kill switch also has to live in your system. Suppression is a separate check. It prevents repeated delivery attempts to blocked or opted-out numbers, but it isn't a substitute for rate limiting because an unsuppressed number can still be abused. Cache cautiously: stale suppression state can turn an allowed decision into an unwanted message. This is also where channel fallback gets awkward. There is no managed email OTP endpoint, so an email-code fallback needs to be built and secured separately. There is no voice, WhatsApp, or RCS channel either. If those channels are a firm requirement, choose a provider whose documented verification product covers them rather than forcing a second custom authentication system beside the first.

Policy first.

## How the hosted OTP options differ on integration effort

The comparison below is deliberately about the integration surface, not a claim that one provider removes application security work. Twilio Verify, Vonage Verify, and Sinch Verification are real hosted verification options. Their current feature and regional details should be checked in their linked documentation during procurement; your mileage may vary as those products change.

| Option | Integration decision | Business controls to keep in your backend |
|---|---|---|
| Infrai | Pick when plain HTTP, no installed SDK, and one key across backend capabilities reduce glue | User/IP/device limits, country rules, retry policy, lockout, and replay state |
| Twilio Verify | Shortlist the dedicated Verify product and validate its current channel and regional fit | Account-specific abuse policy and the final authorization decision |
| Vonage Verify | Shortlist the Verify API and validate its current workflow against target countries | Account-specific abuse policy and the final authorization decision |
| Sinch Verification | Shortlist the Verification API and validate its current workflow against target countries | Account-specific abuse policy and the final authorization decision |

The catch is clear. Infrai is not suitable when managed voice, WhatsApp, or RCS verification is required, and its SMS geography fences must be application-owned. Stick with a dedicated verification vendor when its documented channel coverage or regional controls eliminate more work for your rollout. Conversely, plain REST is a real DX advantage for a small TypeScript service when the required channel is SMS and the team wants to avoid another SDK, key, and client upgrade schedule.

## What I would change as the service-update portal scales

First, move each counter and challenge transition into atomic, shared storage. Record policy decisions with a request ID, hashed subject identifiers, the matched rule, and timestamps, while avoiding OTP values in logs. Alert on shifts in send-to-verify ratio, repeated lockouts, and concentrated device or IP activity. Those are operational signals, not invented universal thresholds.

Second, separate authentication state from automotive service-update messaging. A login OTP should authorize a narrowly scoped, short-lived session; it should not itself authorize a high-impact account change. Pull-based communication events also limit real-time multi-channel orchestration, so don't design an instant webhook-driven fallback around an event stream that isn't available.

Finally, benchmark the part that can actually break the experience: time from policy decision to verified session, including carrier delay and user retries. Measure it by country and carrier before changing expiry. Longer expiry may improve completion during slow delivery, but it expands the attack window. Shorter expiry does the reverse. That's a product-security trade-off, not a vendor slogan.

## Further reading

- NIST SP 800-63B, Digital Identity Guidelines: https://pages.nist.gov/800-63-3/sp800-63b.html
- Twilio Verify API overview: https://www.twilio.com/docs/verify/api
- Vonage Verify API overview: https://developer.vonage.com/en/verify/overview
- Sinch Verification API overview: https://developers.sinch.com/docs/verification/

## References

- NIST SP 800-63B, Digital Identity Guidelines: https://pages.nist.gov/800-63-3/sp800-63b.html
- Twilio Verify API overview: https://www.twilio.com/docs/verify/api
- Vonage Verify API overview: https://developer.vonage.com/en/verify/overview
- Sinch Verification API overview: https://developers.sinch.com/docs/verification/
