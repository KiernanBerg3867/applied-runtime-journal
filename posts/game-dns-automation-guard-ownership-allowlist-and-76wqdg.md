# Game DNS Automation Guard: Ownership Allowlist and Intent Log

A destructive DNS operation needs an automation guard because the service issuing a delete may not own the game zone it can reach. Customer-owned zones and platform-owned zones can sit behind the same registrar adapter, yet they demand different authority.

**Short answer:** put deletion behind three independent checks: an ownership-scoped allowlist, a durable intent record, and an explicit runtime flag. Default to denial. Log the request before calling the provider adapter, bind the approval to the exact zone and owner, and make the adapter accept a validated command rather than a loose domain string.

No single check is enough. An allowlist can be stale. A flag can leak into a scheduled job. A log written after deletion is only an obituary.

## How should a destructive DNS automation guard use an allowlist?

Registrar credentials answer a narrow question: can this principal call an API? They do not answer who owns `guild-ember.example`, whether a customer delegated only DNS hosting, or whether the zone is still serving game discovery, patch delivery, and player email. That distinction matters during migration. A platform-owned launch zone can follow an internal retirement policy. A customer-owned tournament domain needs authorization from the customer boundary represented in the control plane. I would encode those as two separate ownership values, not infer them from which provider currently hosts the zone. Hosting changes. Authority should not drift with it. DMARC makes one hidden dependency especially easy to miss: a domain can publish policy and reporting addresses through DNS, so deleting the zone is wider than removing game traffic. RFC 7489 defines DMARC records at `_dmarc` and describes aggregate and failure reporting destinations. A zone inventory that looks only at `A` and `CNAME` records is incomplete.

The trap is seductive: `deleteZone(name)` is fantastic time-to-first-call. It is also a bad destructive interface. The caller can skip context because the type never asks for any.

## The smallest guard I would ship

Keep provider code at the edge. The policy function should be boring, deterministic TypeScript, with no vendor-specific identifiers in its input. This version requires a ticket-like intent ID, an allowlist entry keyed by both owner and canonical zone, and a process flag that must equal a precise value.

```ts
type Ownership = "customer" | "platform";

type DeleteRequest = Readonly<{
  zone: string;
  ownerId: string;
  ownership: Ownership;
  intentId: string;
  requestedBy: string;
}>;

type AllowedRetirement = Readonly<{
  zone: string;
  ownerId: string;
  ownership: Ownership;
  intentId: string;
  expiresAt: string;
}>;

type ValidatedDelete = DeleteRequest & Readonly<{
  validatedAt: string;
}>;

const canonicalZone = (value: string): string =>
  value.trim().toLowerCase().replace(/\.$/, "");

function authorizeDelete(
  request: DeleteRequest,
  allowlist: readonly AllowedRetirement[],
  env: Readonly<Record<string, string | undefined>>,
  now: Date,
): ValidatedDelete {
  if (env.DNS_DESTRUCTIVE_MODE !== "enabled") {
    throw new Error("destructive mode is disabled");
  }

  const zone = canonicalZone(request.zone);
  const approval = allowlist.find((entry) =>
    canonicalZone(entry.zone) === zone &&
    entry.ownerId === request.ownerId &&
    entry.ownership === request.ownership &&
    entry.intentId === request.intentId
  );

  if (!approval || Date.parse(approval.expiresAt) <= now.getTime()) {
    throw new Error("no current retirement approval matches this request");
  }

  return { ...request, zone, validatedAt: now.toISOString() };
}
```

The exact flag value is intentional. Truthy parsing turns values such as `"false"` into a footgun. Expiry is equally important: without it, yesterday's approved migration silently authorizes tomorrow's unrelated cleanup. These three checks add friction, and that is a real trade-off for local development or disposable test zones. For those cases, use the same guard with short-lived, test-only approvals; do not weaken the production path.

The allowlist belongs in a reviewed control-plane store, not beside the script as a growing config file. Config bloat is still bloat when it is YAML. The useful unit is one narrow approval with an owner, ownership class, intent, zone, and deadline.

## Record intent before crossing the boundary

Authorization and execution should be two events. First append an immutable intent record. Then invoke a small adapter with the validated value. If recording fails, stop. If the provider call fails, retain the attempted state and error category for reconciliation; do not manufacture success.

```ts
type IntentLog = {
  append(event: Readonly<{
    action: "dns.zone.delete.requested";
    zone: string;
    ownerId: string;
    ownership: Ownership;
    intentId: string;
    requestedBy: string;
    validatedAt: string;
  }>): Promise<void>;
};

type ZoneAdapter = {
  deleteValidatedZone(command: ValidatedDelete): Promise<void>;
};

async function retireZone(
  request: DeleteRequest,
  allowlist: readonly AllowedRetirement[],
  env: Readonly<Record<string, string | undefined>>,
  log: IntentLog,
  adapter: ZoneAdapter,
  now = new Date(),
): Promise<void> {
  const command = authorizeDelete(request, allowlist, env, now);

  await log.append({
    action: "dns.zone.delete.requested",
    zone: command.zone,
    ownerId: command.ownerId,
    ownership: command.ownership,
    intentId: command.intentId,
    requestedBy: command.requestedBy,
    validatedAt: command.validatedAt,
  });

  await adapter.deleteValidatedZone(command);
}
```

Notice what is absent: a generic `force` boolean passed through five layers. The runtime switch enables the hazardous code path for an operator-controlled run, while the allowlist authorizes one business object. Those are different jobs. Collapsing them creates a global permission disguised as convenience.

Also absent is a DNS record purge loop. Zone deletion belongs behind one adapter operation because partial record deletion produces an awkward middle state and expands the retry surface. The adapter still needs provider-specific behavior, but the policy does not.

One limitation is deliberate: this guard decides whether execution may start, not whether the zone is safe to retire. Dependency discovery and human approval remain upstream concerns. An allowlist cannot detect an undocumented game client, forgotten mail record, or stale delegation by itself.

Test denials, not just the happy path.

The fastest useful test matrix is mostly rejection cases. I benchmark developer tooling by how quickly a caller reaches a correct first call; for destructive tooling, a correct refusal counts.

| Case | Expected result |
| --- | --- |
| Flag missing or misspelled | Deny before log or adapter call |
| Zone matches, owner differs | Deny |
| Owner matches, ownership class differs | Deny |
| Approval expired | Deny |
| Intent ID differs | Deny |
| Intent append fails | Do not call adapter |
| Every field matches | Append intent, then call adapter once |

Test canonicalization too: uppercase input and a trailing root dot should map to the approved canonical name. Do not add fuzzy matching, suffix matching, or wildcard customer approvals. `play.example` is not authority over `example`, and a helpful matcher is dangerous here.

For observability, count denials by reason without putting credentials or full provider responses into logs. Alert on repeated denied attempts, approvals nearing expiry, requested events with no terminal execution state, and adapter retries. The intent ID is the join key across those signals. Keep owner and ownership class as separate dimensions because they answer different operational questions.

## What I would change at scale

The in-memory array is the first thing to replace. A durable store should enforce uniqueness for the approval tuple and support atomic consumption when policy permits only one attempt. Multiple workers also need an idempotency contract at the adapter boundary so a timeout does not turn a retry into ambiguous repeated work.

I would add a preview stage that captures the zone's record inventory and dependencies, then require a second principal for customer-owned zones. The preview is evidence, not authorization. It should include mail-related records such as DMARC alongside game endpoints, and its age should be bounded so reviewers are not approving a stale picture.

Platform-owned zones can use a shorter internal approval path, but they should still pass the same technical guard. Maintaining two deletion implementations buys a little convenience and doubles the dangerous surface. Keep one execution path; vary the approval policy upstream.

There is a cost. Durable intent storage, review state, and reconciliation take more work than a provider SDK call. The payoff is not an abstract claim of safety. It is a concrete answer to four questions after any run: who requested deletion, which owner authorized it, what exact zone was approved, and whether execution crossed the provider boundary.

Ship the denial path first. Then make the delete call dull.

## Sources

- https://datatracker.ietf.org/doc/html/rfc7489
