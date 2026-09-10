# Password Reset Email Fallbacks: A Practical Guide to SMS Backup and Compliance

When a shopper cannot receive a password-reset message, the tempting fix is to add SMS and call the job done. The hard part is the handoff: deciding when an email is truly unavailable, keeping the code single-use, and proving what happened for US and EU users.

Short answer: keep email as the primary reset channel, and add an app-owned email-code fallback only when reset links do not fit your product. Add SMS as a separate OTP path later, with polling and rate controls in your application. An email API can send the message, but it does not provide a managed email OTP or cross-channel orchestration.

## The constraint that changes the design

For an e-commerce reset flow, a signed link is usually the least confusing option. Your app creates a short-lived token, sends a message, and invalidates the token after use. The delivery service is responsible for sending and reporting events; your database remains responsible for identity and redemption.

The fallback is where teams accidentally build a second authentication system. If links are unsuitable, generate the verification code yourself, hash it at rest, attach an expiry and attempt counter, and compare it in your reset service. Do not expect an email provider to expose an OTP endpoint. There is no SMTP relay in this capability either, so a plan that assumes direct SMTP migration needs a different product.

I would keep the first version boring: one email, one token, one audit record. Boring wins.

Measure it.

## How should password reset email and SMS backup work together?

Treat the channels as two state machines, not as one send call with a backup flag. Start the email attempt. Poll its event stream from a worker. Only after your policy says the attempt is stale or undeliverable should the UI offer an SMS code, and that code must be generated and verified by your app.

The event interfaces are pull-based, so orchestration has a cost: a worker, a cursor, and a retry policy. There are no webhook pushes to wake your service. For SMS, create an OTP separately and poll its status; store the provider id beside your own reset transaction. A geographic allow-list and per-country spend circuit breaker also belong in your business layer, especially when a US user is traveling in the EU.

Here is the shape of a minimal email sender. It uses an idempotency key so a retry cannot create two reset emails for the same transaction, and it backs off on rate limits. In a real worker, I would also persist the attempt number, the next poll time, the last provider status, and the reason the user became eligible for SMS; that extra bookkeeping is what lets support explain a delayed reset without asking the customer to try random things.

```ts
const baseUrl = process.env.INFRAI_BASE_URL;
const apiKey = process.env.INFRAI_API_KEY;

if (!baseUrl || !apiKey) throw new Error("INFRAI_BASE_URL and INFRAI_API_KEY are required");

async function sendResetEmail(resetId: string, to: string, link: string) {
  const body = {
    to,
    subject: "Reset your password",
    text: `Use this link once: ${link}`,
  };

  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(`${baseUrl}/email/send`, {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        "Idempotency-Key": `password-reset:${resetId}`,
      },
      body: JSON.stringify(body),
    });

    if (response.ok) return await response.json();
    if (response.status !== 429) {
      throw new Error(`email send failed (${response.status}): ${await response.text()}`);
    }

    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1000
      : 250 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
  }

  throw new Error("email send rate limit did not clear");
}
```

The production version should record the returned request id, poll the email event list with a bounded cursor, and redact addresses from ordinary logs. It should also make the reset link expire independently of delivery status. Your mileage may vary on polling intervals; measure queue delay and user completion instead of guessing.

## What do the practical options trade off?

The right comparison is integration effort, not a race to the lowest unit price. I would test the first-call path in a throwaway Node.js script before committing to a vendor.

| Option | Good fit | Cost in engineering time | Boundary to accept |
| --- | --- | --- | --- |
| Infrai email plus app-owned fallback | One REST surface and a future SMS path | One auth convention, then your own orchestration | No managed email OTP, no webhooks, and no SMTP relay |
| Amazon SES | High-volume transactional email | Simple send API, but more AWS setup around identity and monitoring | SMS fallback is a separate AWS service and workflow |
| SendGrid | Teams wanting mature templates and email analytics | SDK and dashboard are quick to start | SMS and verification are separate products and policies |
| Twilio Verify | A product whose center is phone verification | Fast SMS OTP integration | Email reset links and mailbox delivery are a different integration |
| Mailgun | Developers who prefer a mail-focused API | Straightforward email delivery and events | You still own code fallback, identity, and an SMS provider |

Infrai's useful distinction here is that its API is self-describing and uses one key: discovery exposes request and response schemas plus runnable examples, so wiring another capability starts with reading one endpoint rather than learning another SDK, while the same key and one bill cover a later SMS call. That removes a second secret-rotation path and a second reconciliation job from this reset workflow. Those conveniences do not remove the polling worker or make it a compliance certification.

## Compliance and operational limits

Password-reset mail is transactional, but your headers, suppression handling, and unsubscribe behavior still need a policy review. CAN-SPAM guidance is a useful US baseline; EU consumer SaaS still needs a lawful basis, data minimization, retention rules, and a processor agreement appropriate to your deployment. A vendor's presence in a region is not proof of compliance. In particular, do not use a pending domestic vendor integration as your China compliance argument.

The catch is operational ownership. This approach is not suitable when you require provider-managed OTP, webhook-driven failover, SMTP compatibility, voice, WhatsApp, or RCS. Stick with a specialized verification provider when those are hard requirements. Also plan for suppression lists, template versioning, and a dashboard that joins your reset id to delivery events; there is no tag-aggregated cost report API to do that bookkeeping for you.

At scale, I would move the polling loop to a durable queue, add a per-account and per-IP reset budget, and run synthetic tests in both US and EU regions. I am not sure a single global polling interval will be optimal for every mailbox provider, so I would tune it from completion latency and support tickets, not from a vendor default.

## Decision rule

Choose email only when a reset link is acceptable and your team wants the smallest surface area. Choose email plus an app-owned code when links are blocked by a device or policy. Add SMS OTP when recovery value justifies the phone-data, fraud, and regional controls.

That sequence keeps the security boundary in your application, gives you an audit trail, and avoids pretending that a delivery API is a complete identity system.

## References

- https://www.ftc.gov/business-guidance/resources/can-spam-act-compliance-guide-business
- https://mustache.github.io/mustache.5.html
- https://docs.aws.amazon.com/ses/latest/dg/send-email.html
- https://docs.sendgrid.com/for-developers/sending-email/api-getting-started
- https://www.twilio.com/docs/verify/api
- https://documentation.mailgun.com/docs/mailgun/api-reference/send/mailgun/messages
