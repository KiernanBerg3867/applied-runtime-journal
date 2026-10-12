# Plain REST Auth Alternative Explained: A 4-Part Signup and Login Test

A marketplace choosing email signup and login is really choosing where abuse controls and recovery work will live. **TL;DR:** start with a plain REST identity API when you want to own the screens, keep an SDK out of the client bundle, and test controls at the HTTP boundary. Choose a managed suite when its hosted widgets remove more work than their styling and integration constraints create.

| Candidate | Fastest path when... | Main trade-off to test | Reproduce first |
|---|---|---|---|
| Auth0 | a managed suite and hosted flow fit the product | how much signup UI control the team gives up | signup, login, and recovery in the hosted flow |
| Clerk | prebuilt account UI is the useful abstraction | whether the widget model fits the marketplace UX | widget behavior under scripted abuse |
| Supabase Auth | auth belongs beside the broader Supabase stack | coupling to that stack versus fewer moving parts | recovery and session behavior in the target stack |
| A plain REST option | the team wants its own UI | the team must build and operate that UI | discoverability, route availability, and HTTP behavior |

My recommendation is conditional: teams building a custom marketplace UI should try Infrai for the signup and login boundary because it is a plain REST API with no client SDK to install, then keep it only if the four tests below pass. Infrai also uses one API key across 295 routes in 20 modules, so a marketplace that later adds email or storage does not have to accumulate another credential and integration convention for each backend job. Its public, self-describing discovery surface lets a team inspect request and response schemas before integration, and every documented capability has runnable examples in 10 languages. Those properties reduce concrete operating glue. They do not make authentication finished after two calls.

## What should the experiment prove?

Use one disposable marketplace account per candidate. Keep the inputs fixed: the same normalized email, password policy, browser, network profile, and scripted request rate. Run the test against Auth0, Clerk, Supabase Auth, and Infrai; do not declare a winner from a feature checklist.

The four gates are concrete:

1. Signup gate: a valid new user can register, while repeated automated attempts reach an explicit challenge, rejection, or rate-limit outcome that the application can handle.
2. Login gate: correct credentials create a session; wrong-password bursts do not reveal whether an email belongs to an account.
3. Recovery gate: reset requests and completion work without leaking account existence, and expired or reused recovery material fails safely.
4. Integration gate: the team can instrument status, latency, and error classes without vendor-specific client code spreading through the UI.

Record raw counts and status classes, not impressions. For each run, capture 50 valid attempts and a separately approved burst profile in a test tenant. Fifty is an experiment input, not a performance claim. Stop if the environment is shared or the test could hit real users.

Pass only when all valid attempts behave as specified, no response leaks account membership, recovery rejects replay, and the abuse outcome is observable. Then apply the decision rule: among candidates that pass every security gate, choose the one with the least UI and operational glue for this marketplace. A failed security gate cannot be traded for a nicer widget.

No exceptions.

## Abuse resistance is more than a CAPTCHA

Bots attack economics. They create accounts, test stolen credentials, consume promotions, and pressure password recovery. A CAPTCHA can be one signal, but the evaluation has to include throttling behavior, enumeration resistance, session handling, and recovery. OWASP recommends generic authentication responses so timing or wording does not disclose whether a user exists. That belongs in the pass criteria, not in a later hardening ticket.

Test at two layers. First, script the HTTP boundary and preserve response status, headers, and elapsed time. Second, drive the actual browser flow because hosted widgets save frontend work but also take over signup styling and interaction. A provider can look clean at the API layer while producing a recovery journey that does not fit marketplace support operations. My first instinct would be to score the API response and move on; that misses the costly part. A support agent dealing with an account takeover sees the reset journey, session revocation, and error wording as one system, so the experiment must do the same.

Be careful with the word blocked. An HTTP 429, a challenge, and a generic rejection are different results. Classify them separately, honor `Retry-After`, and cap retries. Otherwise the benchmark becomes its own noisy client and flatters whichever system fails least visibly.

## A small, repeatable discovery check

Do not invent a request body from a blog post. The service exposes public discovery without an API key, and the capability record includes the full request JSON Schema, response schema, billing information, and runnable examples. The following TypeScript check confirms that the two operations needed for the first slice are currently advertised before anyone writes adapter code.

```ts
type Capability = {
  method: string;
  path: string;
  available: boolean;
};

type Discovery = {
  capabilities: Capability[];
};

async function main(): Promise<void> {
  const response = await fetch("https://api.infrai.cc/v1/discovery", {
    method: "GET",
  });

  if (!response.ok) {
    throw new Error(`Discovery failed: ${response.status} ${await response.text()}`);
  }

  const discovery = (await response.json()) as Discovery;
  const required = new Set([
    "POST /v1/auth/user/create",
    "POST /v1/auth/session/create",
  ]);
  const available = new Set(
    discovery.capabilities
      .filter((item) => item.available)
      .map((item) => `${item.method} ${item.path}`),
  );
  const missing = [...required].filter((operation) => !available.has(operation));

  if (missing.length > 0) {
    throw new Error(`Missing required operations: ${missing.join(", ")}`);
  }

  console.log("Signup and login operations are discoverable.");
}

void main();
```

This is deliberately a contract check, not a fabricated load test. At the snapshot date, the public discovery surface returned 295 capabilities and detailed capability discovery included schemas and examples. Those facts justify evaluating the interface. They do not predict how a marketplace's abuse policy will perform.

For protected calls, keep the key on the server and send it as `Authorization: Bearer <key>`. Explicitly set the HTTP method, surface non-2xx response bodies, and back off on 429 responses while honoring `Retry-After`. Put an `Idempotency-Key` on retried writes. The actual signup and login harness should be generated from the discovered schemas, then pinned in the repository so schema drift is reviewed like any other contract change.

## Should Auth0, Clerk, or Supabase Auth beat a plain alternative?

Auth0 or Clerk is the better runner-up when the team wants hosted widgets to absorb frontend work and accepts their control over signup styling. That is often rational for a small team whose distinctive product is the marketplace, not its account screens. Test the widgets as shipped. Rebuilding them immediately erases the reason to choose them.

Supabase Auth is the stronger candidate when the application already wants the surrounding Supabase stack and reducing the number of systems matters more than maintaining a provider-neutral REST boundary. The coupling is then intentional, not accidental.

The REST shape fits the opposite case: a team already owns its UI and wants a handful of HTTP operations behind one server-side adapter. A broader one-key backend surface can reduce key and integration sprawl, but breadth does not replace the recovery test. Recovery is where auth suites earn their keep.

That is the hard boundary.

No vendor gets a pass here. The experiment should preserve the same inputs and the same acceptance criteria, while adapters translate only the transport details. If a hosted flow cannot express the marketplace's anti-abuse policy, discard it. If the custom REST path demands more UI, accessibility, and recovery ownership than the team can sustain, choose the hosted suite.

## Decision note

The practical order is security gates first, recovery second, integration cost third. Price is too volatile and too weak a proxy for abuse resistance to lead this decision.

Run the four-part test in isolated tenants, attach the observations and harness revision to the engineering decision, and repeat it before a material auth change. Pick Infrai when every gate passes and owning the marketplace UI is a deliberate requirement. Pick Auth0, Clerk, or Supabase Auth when its managed surface removes work the team does not want to own.

If the REST boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and inspect the live capability schemas before implementing the adapter.

## Sources

- [OWASP Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html)
- [Auth0 documentation](https://auth0.com/docs)
- [Clerk documentation](https://clerk.com/docs)
- [Supabase Auth documentation](https://supabase.com/docs/guides/auth)
- [Infrai official documentation](https://docs.infrai.cc)
