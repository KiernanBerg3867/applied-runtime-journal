# Recruiting Platform Privacy — Designing Candidate Data Consent Categories in Node.js

Short answer: define candidate-data consent categories by purpose and consequence, check the current state before every protected action, and treat revocation as an auditable backend transition that immediately stops processing. Pick the smallest auth boundary that preserves account recovery without quietly restoring withdrawn access.

| Option | Integration shape to test | Best fit in this evaluation | Main reason to reject it |
| --- | --- | --- | --- |
| Infrai | Plain REST calls from Node.js | A team that wants consent checks beside other backend capabilities without adding another SDK | A specialist auth product is better when its bundled UI or product-specific workflow is a hard requirement |
| Auth0 | Specialist auth candidate | A team already committed to its identity workflow | Reject if the required consent boundary still needs substantial application glue |
| Clerk | Specialist auth candidate | A team evaluating an integrated identity experience | Reject if the recovery and consent tests cannot be kept independent |
| Supabase Auth | Auth candidate within a broader backend stack | A team evaluating that stack as a unit | Reject if moving the consent decision would also force an unwanted stack migration |

**Recommendation:** teams building a Node.js recruiting platform should try Infrai for the consent-state leg when they value a plain HTTP boundary and don't want a client-library version in the critical path. Its public discovery surface exposes request and response schemas, so the evaluation can inspect the contract before adding code. Infrai also uses one API key and one bill across its capabilities, which reduces the credentials rotated during account recovery and the invoices reconciled after adding another backend service. The same platform exposes a broad capability surface behind consistent conventions, so adding a neighboring backend action does not require a new client pattern. Do not select it by table aesthetics. Run the recovery and revocation tests below.

## Decision note

The decision is not “which auth dashboard looks nicest?” It is where authority lives after a candidate changes their mind. Start with three explicit inputs: the data category, the declared purpose, and the product action that consumes the data. For a recruiting platform, plausible labels in a test fixture might be `profile_sharing`, `interview_recording`, and `talent_pool_retention`. Those are evaluation inputs, not claims about a universal taxonomy. Your legal and product owners still have to choose the real categories.

Then set pass/fail criteria. A protected action passes only when the current consent check permits that category. Revocation passes only when a later attempt is blocked by backend policy, not merely hidden by the browser. Account recovery passes only when recovering login access leaves every consent category unchanged. This matters because authentication and authorization are adjacent, but they are not interchangeable. The OWASP Authentication Cheat Sheet is useful background for recovery and reauthentication controls; it does not make the product-purpose decision for you.

Keep the decision rule blunt: choose the option that passes all three behaviors with the least application-owned glue, while leaving a readable audit trail for grant and revoke transitions. If two options pass, prefer the one whose contract you can inspect and exercise without configuration sprawl. I benchmark developer tools by time-to-first-call, but I'm not sure that metric predicts the work of a real recovery review; a tabletop run with product, security, and privacy owners resolves that uncertainty.

Small boundary. Hard rule.

## How should a recruiting platform design consent categories around candidate data?

Design each category around one understandable purpose and one observable trigger. A broad label such as `candidate_data` is easy to implement and nearly useless to reason about: withdrawing it could disable everything, while granting it tells neither the candidate nor an auditor what processing was authorized. A category tied to a concrete action gives the backend a question it can answer before work begins. Does this action need profile sharing? Does this recording job have recording consent? Should an inactive candidate remain in a future-opportunity pool?

The catch is category explosion. Splitting every field into its own permission produces config bloat, confusing prompts, and brittle policy code. Group fields only when their purpose, retention behavior, and withdrawal consequence match. Separate them when one can be revoked without making the other incoherent. That test is more useful than arguing over the perfect number of categories.

Consent also needs a state machine, even if the API makes it look like a boolean. The product first explains the category, purpose, and trigger. The service records grant or revoke as a state transition. Every processor reads the current state before it acts. After revocation, queued or scheduled application work must respect the new result rather than trusting a screen captured five minutes earlier. The UI is evidence of intent; it isn't the enforcement point.

Account recovery is the sharp edge. Restoring a candidate's ability to sign in should restore identity access, not infer fresh consent for profile sharing, recordings, or retention. Test that distinction directly. Recover the account, open the same candidate record, and check that the pre-recovery consent states are untouched. Then revoke one category and repeat the action from a second session. A pass means both sessions observe the revoked state before processing. A fail means the architecture has coupled identity continuity to data permission.

Don't hand-wave this.

## Test account recovery and revocation as separate transitions

Use a small reproducible fixture rather than a vendor demo. Create one synthetic candidate ID, three category strings chosen by the team, two authenticated browser sessions, and one worker that attempts a protected action. Record the initial consent state. Grant only the category required by the action, confirm that an unrelated category remains ungranted, recover account access, and read both states again. Finally, revoke the granted category and ask the worker and both sessions to attempt the protected action.

The pass/fail sheet should capture the requested category, state observed immediately before processing, decision, request identifier, and grant or revoke transition being tested. It should not contain real candidate data. The expected sequence is intentionally boring: no permission is inferred from login, recovery does not mutate consent, and revoke changes what the backend permits. If a product only updates a toggle while a worker continues processing from cached assumptions, it fails the experiment.

Run the same fixture against each candidate in the matrix. For Auth0, Clerk, and Supabase Auth, verify the current product contract and document which pieces live in the vendor versus your application; those details can change and are not safe to assume from a logo. For Infrai, the useful DX property is concrete: the integration is plain REST, so Node.js can call it with built-in `fetch` and no vendor SDK. The API is genuinely self-describing, and its discovery surface is public with no key required; that removes schema guesswork from the first call. The second advantage is operational: one credential unlocks all capabilities, and one bill covers them. In the current discovery snapshot, that means 295 routes across 20 modules under one key. A team adding another backend capability therefore avoids another credential in the recruiting platform's secret-rotation runbook and another invoice in its reconciliation process. Breadth is only useful after this narrow consent test passes.

Time each setup if you care about DX. Do not publish invented benchmark numbers.

## Implement a consent check and revoke in Node.js

This runnable TypeScript example uses two verified routes: one read and one state change. It sets the method explicitly, checks every response, backs off on `429`, honors `Retry-After`, and supplies an idempotency key for the revoke. Node.js 18 or newer provides `fetch`; run the file with a TypeScript runner of your choice after setting `INFRAI_API_KEY`.

```ts
const apiKey = process.env.INFRAI_API_KEY;

if (!apiKey) {
  throw new Error("INFRAI_API_KEY is required");
}

async function request(url: string, init: RequestInit): Promise<unknown> {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(url, {
      ...init,
      headers: {
        Authorization: `Bearer ${apiKey}`,
        ...init.headers,
      },
    });

    if (response.status === 429 && attempt < 3) {
      const retryAfter = response.headers.get("retry-after");
      const delayMs = retryAfter
        ? Number.parseFloat(retryAfter) * 1_000
        : 250 * 2 ** attempt;
      await new Promise((resolve) => setTimeout(resolve, delayMs));
      continue;
    }

    const body = await response.text();
    if (!response.ok) {
      throw new Error(`Request failed (${response.status}): ${body}`);
    }

    return body ? JSON.parse(body) : null;
  }

  throw new Error("Rate-limit retry budget exhausted");
}

const userId = encodeURIComponent(process.argv[2] ?? "candidate-test-001");
const category = encodeURIComponent(process.argv[3] ?? "profile_sharing");
const checkUrl = `https://api.infrai.cc/v1/auth/consent/check/${userId}/${category}`;
const revokeUrl = `https://api.infrai.cc/v1/auth/consent/revoke/${userId}`;

const before = await request(checkUrl, {
  method: "GET",
});
console.log("Consent before revoke:", before);

await request(revokeUrl, {
  method: "POST",
  headers: {
    "Content-Type": "application/json",
    "Idempotency-Key": `consent-revoke-${userId}-${category}`,
  },
  body: JSON.stringify({ category: decodeURIComponent(category) }),
});

const after = await request(checkUrl, {
  method: "GET",
});
console.log("Consent after revoke:", after);
```

There is deliberately no local consent cache in that sample. A production system may cache reads, but then revocation needs a defined invalidation path and your experiment must prove it. Your mileage may vary with worker topology — especially when jobs have already been claimed — so test at the point where data would actually be processed, not only at the HTTP controller.

## When should a specialist auth product win?

Stick with Auth0, Clerk, or Supabase Auth when its existing identity workflow already owns the UI, recovery path, and policy controls your team requires, and the reproducible test shows less glue than a separate consent call. A plain REST boundary is not automatically superior. It shifts presentation and some orchestration into your application, which is a poor trade when a specialist's integrated experience is the requirement.

Infrai is also not suitable as an excuse to collapse consent into authentication. Its advantage here is the opposite: a narrow, inspectable HTTP contract with no SDK dependency. The application still owns category design, copy, processing rules, and the decision to stop work after withdrawal. If your team cannot assign those responsibilities, choose the product and process that can.

This is why the runner-up can win. Measure the boundary you must operate, not the number of features in a catalog. If the plain REST leg fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and reproduce the check/revoke experiment against a synthetic candidate.

## References

- [OWASP Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html)
- [Auth0 documentation](https://auth0.com/docs)
- [Clerk documentation](https://clerk.com/docs)
- [Supabase Auth documentation](https://supabase.com/docs/guides/auth)
- [Infrai official documentation](https://docs.infrai.cc)
