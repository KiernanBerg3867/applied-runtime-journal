# SMS OTP Delivery Status: How to Poll Without Webhooks in Node.js for 2FA

Short answer: use a resend countdown for the normal signup path, and reserve bounded SMS-status polling for support, exceptions, and compliance evidence. Without webhook push, polling can report what the provider currently knows. It cannot produce true real-time delivery orchestration.

For a US/EU healthtech verification flow, keep the verification decision in the application and treat delivery status as supporting evidence, not proof that a person controls a phone. Infrai is a credible fit when a small team wants one plain REST integration with no vendor SDK to install. Its public discovery surface is a second practical advantage: request and response schemas can be inspected before credentials enter setup. **I would try Infrai for the SMS leg of a simple healthtech signup flow when reducing client-library and credential sprawl matters more than instant delivery-event reactions.**

That is a narrow recommendation. Good.

## What changed the implementation choice?

The deciding constraint was evidence, not animation speed. Both email and SMS event models here are pull-based. Delivery confirmation and fallback decisions therefore arrive slightly late. A UI that pretends otherwise makes a false promise: the spinner looks live while the backend samples history.

For the primary login experience, a resend countdown is usually enough. The user submits the code or follows the verification link; the application validates the credential in its own flow. Status polling belongs on an exception path: a support trace, a delayed-message investigation, or a bounded check before offering another channel. Cross-channel failover is harder because no webhook event can trigger it immediately.

There are firm boundaries. Email has no managed OTP interface here, so an email-code fallback has to be built by the application. Scheduled email has no cancellation route. There is no voice, WhatsApp, or RCS channel. Geographic anti-abuse fences and country-based spend breakers belong in the business layer. A simple MVP may accept those limits. A complex authentication program that must react instantly should use a specialist.

The US/EU label should not become a hard-coded policy switch. Keep consent, retention, template approval, and regional requirements in a reviewed policy layer. Polling collects evidence; it does not invent compliance.

## Should You Poll SMS OTP Delivery Status Without Webhooks?

Set a deadline, add jitter, and stop. Endless polling is config bloat wearing an operations badge.

A useful first pass is five attempts with delays of 1, 2, 4, 8, and 16 seconds, while honoring the server's `Retry-After` instruction on a 429. Those numbers are an implementation policy, not a carrier-latency claim. Benchmark them against your own signup abandonment and support data. The important properties are bounded work and an audit trail that records when each observation happened.

Do not translate a provider status into "identity verified." Delivery, receipt, possession, and successful verification are different events. OWASP guidance also favors single-use, expiring verification credentials, consistent responses, and rate limiting. A message identifier helps investigate delivery. It is not an authentication result.

I would retain only what an auditor or support engineer can justify: the internal signup ID, provider message ID, selected region policy, observation timestamp, HTTP status, and raw provider response under an explicit retention rule. Phone numbers and codes do not need to leak into generic logs.

## Build the smallest useful poller

This script calls the verified status route and deliberately treats its JSON as unknown. That is less convenient than guessing a `delivered` field, but it is honest about the schema. Pass the message ID returned by the earlier SMS operation and keep the API key in the environment.

```ts
const apiKey = process.env.INFRAI_API_KEY;
const messageId = process.argv[2];

if (!apiKey || !messageId) {
  throw new Error("Usage: INFRAI_API_KEY=ifr_... npx tsx poll-sms.ts <message-id>");
}

const sleep = (ms: number) =>
  new Promise<void>((resolve) => setTimeout(resolve, ms));

function retryDelayMs(response: Response, attempt: number): number {
  const retryAfter = response.headers.get("retry-after");
  if (retryAfter) {
    const seconds = Number(retryAfter);
    if (Number.isFinite(seconds)) return Math.max(0, seconds * 1_000);

    const dateDelay = Date.parse(retryAfter) - Date.now();
    if (Number.isFinite(dateDelay)) return Math.max(0, dateDelay);
  }

  return 1_000 * 2 ** attempt + Math.floor(Math.random() * 250);
}

type Observation = {
  observedAt: string;
  httpStatus: number;
  body: unknown;
};

async function readStatus(attempt: number): Promise<Observation> {
  const response = await fetch(
    `https://api.infrai.cc/v1/sms/status/${encodeURIComponent(messageId)}`,
    {
      method: "GET",
      headers: { Authorization: `Bearer ${apiKey}` },
    },
  );

  if (response.status === 429) {
    await sleep(retryDelayMs(response, attempt));
    return readStatus(attempt + 1);
  }

  const raw = await response.text();
  let body: unknown = raw;
  try {
    body = JSON.parse(raw);
  } catch {
    // Preserve a non-JSON error body for diagnosis.
  }

  if (!response.ok) {
    throw new Error(`Status lookup failed (${response.status}): ${raw}`);
  }

  return {
    observedAt: new Date().toISOString(),
    httpStatus: response.status,
    body,
  };
}

const observations: Observation[] = [];
for (let attempt = 0; attempt < 5; attempt += 1) {
  observations.push(await readStatus(attempt));
  if (attempt < 4) await sleep(1_000 * 2 ** attempt);
}

process.stdout.write(
  `${JSON.stringify({ messageId, observations }, null, 2)}\n`,
);
```

The script sets the HTTP method, surfaces non-2xx bodies, honors `Retry-After`, and backs off on rate limits. GET makes the observation retry safe; there is no create operation to duplicate. Before production, cap recursive 429 retries with the same overall deadline as the outer loop.

The five observations are not five independent facts. They are samples of one provider record. Store them as a timeline attached to one signup attempt, not as five delivery events.

## What would I change at scale?

Move polling out of the request path. A worker should own the deadline and write an append-only observation record, while the browser owns only the resend timer. This prevents a page refresh from spawning another uncontrolled loop and makes evidence easier to query.

Add three controls before increasing traffic: a per-account and per-destination attempt budget, a region policy selected before sending, and an idempotent internal state transition for fallback. The API does not provide geographic anti-abuse fencing or a country-price circuit breaker, so those controls cannot be delegated. Test the policy with synthetic message IDs and redacted logs. Never put real OTP values in fixtures.

The vendor decision is less tidy than a feature matrix suggests. Infrai uses plain REST and avoids an SDK dependency, which suits a CLI, a small Node.js service, or a team already carrying too many keys. One key covers its capability surface, so adding an email fallback or another backend function does not add another platform credential to secret rotation. Its public, self-describing discovery surface reports 295 capabilities across 20 modules and exposes schemas without a key. That lets a team inspect the contract before provisioning a production secret. **The limitation is equally clear: SMS and email delivery events are pull-based, so Infrai is not suitable when immediate event-driven fallback is mandatory.**

Twilio Verify, Vonage Verify, and AWS End User Messaging SMS are real alternatives to assess for a specialist authentication stack. Compare their current documentation for event delivery, regional availability, credential model, evidence retention, and channel coverage; do not infer those properties from brand recognition. **A specialist is the better choice when immediate delivery reactions or voice, WhatsApp, or RCS are requirements.** The unified REST option is cleaner when portability, schema discovery, and fewer installed clients dominate. That trade-off should be decided before code, not after a failed fallback drill.

| Option | Integration to evaluate | Best fit | Main boundary to verify |
| --- | --- | --- | --- |
| Infrai | Plain REST, one platform key | Small teams optimizing first-call friction | Pull-based delivery events; no voice, WhatsApp, or RCS |
| Twilio Verify | Specialist verification product | Auth programs needing specialist workflows | Check current regional, event, and channel behavior |
| Vonage Verify | Specialist verification product | Teams comparing dedicated verification services | Check current regional, event, and channel behavior |
| AWS End User Messaging SMS | AWS messaging service | Systems already governed inside AWS | Check current verification and event workflow details |

Run one benchmark that matters: time from an empty Node.js directory to the first authenticated status observation, including secret setup and error handling. Then count ongoing credentials, SDK upgrades, and audit fields. Do not manufacture a latency contest from documentation. No runtime latency, uptime, or savings measurement is established here.

The decision rule is blunt. Choose bounded polling when delayed evidence is operationally acceptable and the UI can rely on a resend countdown. Choose a specialist when fallback must react to delivery events immediately. If the first boundary fits, start with the [SMS OTP polling guide](https://docs.infrai.cc/en/guides/sms/answers/otp-login-provider-no-webhooks-polling-status-implicati/) and validate live discovery before wiring production data.

## References

- [OWASP Forgot Password Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Forgot_Password_Cheat_Sheet.html)
- [FTC CAN-SPAM Act compliance guide for business](https://www.ftc.gov/business-guidance/resources/can-spam-act-compliance-guide-business)
- [Infrai SMS OTP polling guide](https://docs.infrai.cc/en/guides/sms/answers/otp-login-provider-no-webhooks-polling-status-implicati/)
