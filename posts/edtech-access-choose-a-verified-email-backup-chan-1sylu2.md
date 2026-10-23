# Edtech Access: Choose a Verified Email Backup Channel for SMS OTP Failure

Short answer: use email as the backup channel only after the SMS attempt has a terminal delivery signal, and only when the account already has a verified email address. Do not treat a timeout as proof that a phone number is invalid. Keep the OTP one-time, short-lived, rate-limited, and bound to the login attempt.

| Channel decision | Evidence required | Next action |
| --- | --- | --- |
| SMS still pending | Provider or carrier has not declared failure | Wait, then offer a controlled retry |
| SMS rejected or undeliverable | A typed failure category and recorded attempt | Permit verified-email fallback |
| Email unavailable or unverified | No second trusted address | Stop and use account recovery support |

The recommendation is deliberately boring: make the state machine strict, and make the fallback small. In an education product, a learner missing a class because a carrier delayed one message is a reliability problem. Sending a second OTP to an unverified address is an account-takeover problem. Those are different incidents.

The transport is not the policy.

Keep it boring.

## How should an edtech login choose an email backup channel after SMS OTP failure?

A Node.js service should not infer delivery from the absence of a callback. Network delay, handset reachability, filtering, and provider queueing can all look like silence. I initially thought a timeout was a reasonable proxy for failure. It is not. The application needs an explicit result such as `accepted`, `delivered`, `rejected`, `expired`, or `unknown`, with the timestamp and attempt identifier attached. The exact vocabulary can differ by integration. The meaning cannot.

For a school or tutoring platform, the useful rule is this: `rejected` and a confirmed `undeliverable` result can open the email branch; `unknown` cannot. A delayed message may still arrive after the user has requested another code, so every code needs a generation identifier and a server-side validity check. Only the newest valid challenge should succeed.

That is a bad shortcut.

Measure it.

Here is the small boundary I want between transport events and login logic. It avoids a pile of provider-shaped conditionals in the authentication handler.

```ts
type DeliveryState = 'accepted' | 'delivered' | 'rejected' | 'expired' | 'unknown';

type Attempt = {
  id: string;
  state: DeliveryState;
  emailVerified: boolean;
  smsWasRejected: boolean;
};

function canOfferEmail(attempt: Attempt): boolean {
  return attempt.smsWasRejected &&
    attempt.state === 'rejected' &&
    attempt.emailVerified;
}
```

The boolean is intentionally redundant with `state` in this sketch. In production, I would derive it from one normalized event record instead of trusting two independently updated fields. The point is the decision boundary: an email fallback is earned by evidence, not by impatience.

## Is a verified email enough to protect the fallback?

No. It is necessary, not sufficient. The email address must belong to the account before the SMS failure. A user who cannot pass the primary factor should not be allowed to add or replace the backup address during the same recovery flow. That would turn “SMS failed” into “choose your own second factor.”

The fallback challenge should have the same basic controls as the original challenge: cryptographically unpredictable code generation, single use, a short expiry, attempt limits, and a response that does not reveal whether an account exists. OWASP calls out these controls for reset and recovery flows, and they apply cleanly to OTP login. Store a digest or otherwise protected representation of the code where practical; do not put the raw value in ordinary logs.

The session also needs a purpose. A code issued for `login` should not satisfy `change-email`, and a code issued for one browser challenge should not be accepted by another session without an intentional binding rule. The simplest implementation is a server-side challenge record keyed by a random challenge ID, with the account ID, purpose, expiry, attempt count, and delivery channel.

There is a human trade-off here. A five-minute expiry may be frustrating for a learner moving between a phone and a laptop; a long expiry enlarges the window for interception and replay. Pick a short policy, measure expired challenges, and adjust from evidence. Do not silently extend validity after a resend.

## The Node.js flow that stays understandable

The handler should orchestrate a state transition, not decide what carrier errors mean. Normalize the transport event before it reaches authentication. Keep the challenge ID stable across status updates, and create a new challenge only when the product intentionally starts a new attempt.

```ts
type Channel = 'sms' | 'email';

type Challenge = {
  id: string;
  accountId: string;
  purpose: 'login';
  channel: Channel;
  codeDigest: string;
  expiresAt: number;
  attempts: number;
};

function nextChannel(challenge: Challenge, smsState: DeliveryState, emailVerified: boolean): Channel | null {
  const smsTerminal = smsState === 'rejected' || smsState === 'expired';
  if (challenge.channel === 'sms' && smsTerminal && emailVerified) return 'email';
  return null;
}
```

That function does not send anything. It returns a decision that can be audited, tested, and rejected by policy. The sender then creates the email challenge under the same account and purpose, invalidating the older code according to the product’s chosen policy. This separation matters when a webhook arrives after the user has already opened the fallback screen.

Keep the route surface small. One endpoint starts a challenge, one verifies it, and a status consumer records delivery events. More routes do not create more reliability. They create more places to disagree about expiry and attempt limits.

## What should be measured before calling it reliable?

Delivery reliability is not the percentage of OTP requests that were accepted by a sender. That is an upstream acknowledgement. Track the funnel by country or region, carrier category where legally and operationally available, channel, and failure state: requested, accepted, delivered, rejected, expired, verified, and abandoned. In the US and EU, retention and access rules for these records need review with the team responsible for privacy and compliance.

Watch two ratios separately. First, how often SMS reaches a terminal failure that qualifies for fallback. Second, how often a verified email fallback produces a successful login. A rising fallback-success rate can mean the email path works. It can also mean SMS routing has degraded. The two numbers need their own baselines.

Log identifiers, not secrets. A useful event includes a challenge ID, account pseudonym, channel, normalized outcome, timestamp, region, and policy version. It does not need the OTP, the full email address, or a phone number in a dashboard. Add structured reasons for denial: unverified email, non-terminal SMS state, expired challenge, rate limit, or wrong code.

Test the ugly sequence, because that is the normal sequence in production: SMS is delayed, the user requests a resend, the first message arrives, the email branch opens, and an old code is submitted. The expected result should be deterministic. One current challenge wins; stale challenges do not.

## When should email wait?

Email should wait when the SMS outcome is `unknown`, when the account has no pre-verified address, or when rate limits indicate repeated automated attempts. It should also wait when the account policy requires a stronger recovery procedure, such as staff review for a high-risk role. The fallback channel is a continuity mechanism, not an entitlement.

This choice costs some successful logins. That is the point. A loose fallback may improve short-term completion while weakening the account boundary. For an edtech system holding coursework, attendance records, or instructor access, the right optimization target is verified continuity, not raw message volume.

The cleanest implementation is therefore a two-gate policy: a terminal SMS failure, then a pre-verified email. Everything else remains pending, rate-limited, or in recovery. It is less clever than automatic escalation. It is easier to explain during an incident, easier to test with a fake transport, and much harder to turn into an accidental bypass.

## References

- OWASP, “Forgot Password Cheat Sheet”: https://cheatsheetseries.owasp.org/cheatsheets/Forgot_Password_Cheat_Sheet.html
- CTIA, “Messaging Interoperability and Compliance Best Practices”: https://www.ctia.org/the-wireless-industry/industry-commitments/messaging-interoperability-sms-mms
- RFC 6238, “TOTP: Time-Based One-Time Password Algorithm”: https://www.rfc-editor.org/rfc/rfc6238
- NIST SP 800-63B, “Digital Identity Guidelines: Authentication and Lifecycle Management”: https://pages.nist.gov/800-63-4/sp800-63b.html
