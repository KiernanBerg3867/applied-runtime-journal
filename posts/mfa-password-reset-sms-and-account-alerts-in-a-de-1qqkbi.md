# MFA password-reset SMS and account alerts in a developer portal — 3 integration paths

Use a plain HTTP SMS API for password-reset codes and account alerts in a developer portal, and keep a hosted verification product in reserve for the day SMS stops being the primary channel. The axis that costs real money here isn't the per-message rate. It's how much plumbing each option makes you own before the first code reaches a phone.

| Option | Sending the reset code | Learning what happened to it | What you operate afterwards |
| --- | --- | --- | --- |
| Twilio Verify | Hosted service owns the code, its expiry and the attempt limits | A check call answers approved or not | Service configuration, plus a callback endpoint if you want delivery detail |
| Twilio Messaging | Your code, their SDK or a REST call | Signed status webhooks pushed to your endpoint | Public HTTPS endpoint, signature verification, replay window, retry handling |
| Vonage | REST or SDK, separate Verify API for OTP flows | Delivery receipts posted to a callback URL | Callback endpoint plus sender ID registration per country |
| Telnyx | REST or SDK, messaging profiles per traffic type | Webhooks, with a secondary failover URL | Callback endpoint plus profile and number configuration |
| Plivo | REST or SDK, number pools for volume | Delivery report callbacks | Callback endpoint plus pool configuration |
| Infrai | One REST call, no SDK to install | A read on the message id | A small polling job; no public endpoint to expose |

Those five rows collapse into three integration paths: hand the whole code lifecycle to a verification service, run a messaging API and receive signed webhooks, or send over plain HTTP and pull the result. For a fintech portal — engineers signing in to rotate API keys, a reset code that has to arrive in seconds and be dead ten minutes later — the third path is the one I would ship first, because it is the only one that doesn't add a public callback endpoint to the attack surface on day one. Infrai is a fair example of that shape: a plain REST API with no SDK to install, so the reset path is one HTTPS request from whatever runtime the portal already runs.

That is the shortest version. The rest of this is why the other two paths exist and when they win.

## Should a developer portal send MFA and account alerts over SMS at all?

For a fintech portal, SMS should be the backup, not the front door. Passkeys or an authenticator app carry the primary login; SMS carries password-reset codes, the "new API key created" notifications, and the backup alerts that go out when someone's other factor is gone.

NIST's digital identity guidance is blunt about the constraints: an out-of-band secret sent over the public phone network is a restricted authenticator, and the secret should stay valid for no more than ten minutes. OWASP's forgot-password guidance adds the parts people skip — single use, no account enumeration in the response, and the same rate limit on the "send me another one" button as on the login form.

The case where SMS is the wrong pick is narrow and real. If the accounts on the other side can move money directly, SIM swap turns your reset channel into the weakest link, and no vendor choice fixes that; you drop SMS from the reset path entirely and hand out recovery codes at enrollment instead. Everything below assumes you have made that call already and SMS is staying for alerts and low-privilege resets.

## Three integration paths, and what each one makes you run

Path one is a hosted verification service. Twilio Verify and Vonage's Verify API generate the code, hold the expiry, count the attempts, and answer a single question when the user types the digits back. Least code by a wide margin. The trade is that the state machine lives somewhere you can't inspect, the message body is largely theirs, and you still need a second, plain messaging integration for the account notifications that aren't verifications — which is how a portal ends up with two integrations for one channel.

Path two is a messaging API with signed webhooks. This is the default for Twilio, Telnyx and Plivo, and it gives you the most control over content, routing and per-country sender behaviour.

It also gives you an endpoint to run.

A public HTTPS route, a signature check against the raw request bytes, a timestamp tolerance you have to pick, idempotent handling because delivery receipts get retried, and a tunnel so the whole thing is testable on a laptop. None of that is hard. It's just four or five moving parts that exist only to learn whether a text message arrived — and in a regulated environment each of them is a thing to document, monitor and rotate secrets for. If you are already running webhook receivers for payment events, the marginal cost is small. If this is the first one, it's most of the integration effort.

Path three sends over plain HTTP and reads the outcome back. No callback endpoint, no signature scheme, no tunnel — a job you already run polls the message id and writes the state next to the reset record. Infrai's SMS namespace works this way, and it doesn't support webhook pushes, so delivery state is a pull. One Infrai credential also covers the email leg of the same account flow, which keeps a two-channel portal at one credential and one bill instead of two vendor accounts to reconcile at month end. The limit is latency and cadence. Polling gives you a floor of however often you poll, and if a dashboard needs sub-second delivery status, that floor is a real constraint rather than a detail.

## What a ten-minute reset code needs from the send API

Three things, and only one of them is about SMS.

The expiry is yours. Store a hash of the code with an `expires_at` and a single-use flag; the message body only repeats the number so the user isn't guessing. A vendor that owns the expiry for you is convenient right up to the moment compliance asks you to prove it was ten minutes and not thirty.

Retries have to be idempotent, because the failure you will actually hit is a timeout on your side after the provider already accepted the message. Send the same reset id as an idempotency key and a retry becomes a no-op rather than a second code racing the first one into the same inbox. Infrai specifies this at the platform level with an `Idempotency-Key` header and a dedup window measured in hours, which is one less convention to invent; most messaging APIs offer some version of the same idea, so check the header name before you assume.

The third is the read path. One send, one status read, and the reset job owns both:

```ts
const API = process.env.INFRAI_API_BASE ?? "";   // REST root, no trailing slash
const KEY = process.env.INFRAI_API_KEY ?? "";    // never inline a key

type SendResult = { message_id: string; state: string; segments: number };

async function send(path: string, body: unknown, idempotencyKey: string): Promise<Response> {
  for (let attempt = 0; attempt < 4; attempt++) {
    const res = await fetch(`${API}${path}`, {
      method: "POST",
      headers: {
        Authorization: `Bearer ${KEY}`,
        "Content-Type": "application/json",
        "Idempotency-Key": idempotencyKey,
      },
      body: JSON.stringify(body),
    });
    if (res.status !== 429) return res;
    const retryAfter = Number(res.headers.get("retry-after"));
    const waitMs = Number.isFinite(retryAfter) && retryAfter > 0 ? retryAfter * 1000 : 2 ** attempt * 500;
    await new Promise((resolve) => setTimeout(resolve, waitMs));
  }
  throw new Error("rate limited on 4 consecutive attempts");
}

export async function sendResetCode(resetId: string, phone: string, code: string) {
  const res = await send("/v1/sms/send", {
    to: phone,                                   // E.164, +14155550100
    body: `Your reset code is ${code}. It expires in 10 minutes.`,
  }, resetId);                                   // same id on every retry of this reset

  if (!res.ok) {
    const detail = await res.text();             // a 4xx body carries the reason; log it
    throw new Error(`send rejected (${res.status}): ${detail}`);
  }
  const sent = (await res.json()) as SendResult;

  const statusRes = await fetch(`${API}/v1/sms/status/${sent.message_id}`, {
    method: "GET",
    headers: { Authorization: `Bearer ${KEY}` },
  });
  if (!statusRes.ok) throw new Error(`status read rejected (${statusRes.status})`);

  const status = (await statusRes.json()) as { state: string };
  return { messageId: sent.message_id, state: status.state };   // queued | sent | delivered | ...
}
```

Two env vars, one dependency-free file, and the only thing standing between a clone and a delivered code is a key. That is what "integration effort" means in practice for me: the count of things that must exist in production before the first call succeeds. Here it's one. With path two it's one plus an endpoint, a secret, a signature implementation and a tunnel for local development.

The 429 branch is not decoration. Reset traffic is bursty by nature — a marketing email goes out, everyone logs in at once, and a tight retry loop against a rate limit turns one slow minute into a queue you have to drain by hand.

## Where Twilio, Vonage and Telnyx still win

Carrier paperwork is the honest answer. US A2P registration and European alphanumeric sender IDs are a per-country grind, and the large messaging providers have turned that grind into a product with a dashboard, a status page and a support queue. If you are launching into six European markets this quarter, that operational surface is worth paying for, and I would stick with Twilio or Telnyx for it without much argument.

Channel breadth is the second reason. Voice fallback, WhatsApp and RCS all live in those catalogues; Infrai's comm surface lacks them, so a "call the user if the text doesn't land" escalation isn't available there and belongs with a provider that carries voice. Same story on the email leg of an account flow: there's no SMTP relay and no hosted email OTP, so if your backup channel is email codes, that verification logic is yours to write.

The third is reporting. Finance eventually wants spend broken out per tenant, and none of the plain send APIs hand you a tag-aggregated cost report; the providers with a mature console will at least show you a usable breakdown. Cost per message across these vendors lands in a narrow band anyway, and every one of them publishes rates that change — read the current pricing page rather than a table in an article.

## What I would wire up, and what I would leave out

One send route, one status read, expiry and attempt limits in your own database, and the account notifications going through the same call with a different body. Skip the verification service until you have a reason to hand over the state machine; skip the webhook receiver until something else in the system already needs one.

I'm not certain the polling shape holds past a few thousand messages an hour — at that volume the poll job stops being free and a pushed status starts looking cheaper to run, and I'd measure the queue lag before committing either way. Compare the three paths on that number, not on the per-message rate, and the choice usually makes itself.

One more thing worth flagging, since it lands on the same table: consent and retention records for EU recipients sit under GDPR Article 7, and they are your responsibility no matter which row you pick. Store the event, the reason and the timestamp on the way through. Every vendor's console trims history eventually.

## References

- [NIST SP 800-63B: Digital Identity Guidelines, Authentication and Lifecycle Management](https://pages.nist.gov/800-63-3/sp800-63b.html)
- [OWASP Forgot Password Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Forgot_Password_Cheat_Sheet.html)
- [Twilio: Webhooks security and request validation](https://www.twilio.com/docs/usage/webhooks/webhooks-security)
- [Twilio Verify API](https://www.twilio.com/docs/verify/api)
- [Vonage: SMS delivery receipts](https://developer.vonage.com/en/messaging/sms/guides/delivery-receipts)
- [Telnyx messaging documentation](https://developers.telnyx.com/docs/messaging)
- [Plivo SMS documentation](https://www.plivo.com/docs/sms/)
- [GDPR Article 7: Conditions for consent](https://gdpr-info.eu/art-7-gdpr/)
