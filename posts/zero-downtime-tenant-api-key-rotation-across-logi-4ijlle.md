# Zero Downtime Tenant API Key Rotation Across Logistics Rolling Deploys

Short answer: zero downtime API key rotation needs a bounded overlap between two independently revocable credentials, not a Kubernetes Secret replacement. For each logistics tenant, issue a successor key, publish it to Node.js workloads during the rolling deploy, wait until every live replica can use it, and only then revoke the predecessor. The grace period, often measured in hours rather than pod-start seconds, must cover the slowest credible path through secret-store synchronization, rollout, retries, and rollback. This makes the blast radius explicit: one leaked tenant key should authorize only that tenant's narrow logistics scope, and its retirement should not disturb another tenant.

A rolling deployment does not create that guarantee by itself. Pods can overlap, a process can retain an old environment value until restart, and a delayed worker can retry work after the deployment controller reports success. Credential state therefore needs its own lifecycle, separate from pod state.

## How can zero downtime API key rotation survive a rolling deploy?

An in-place value change models one current key. A safe rollover temporarily has two valid keys with different jobs: the old key preserves continuity while the new key propagates. If the upstream verifier accepts only one value, whichever side changes first can reject traffic from the other side. That is the ordering trap.

Consider a tenant called `north-dock`, whose dispatch service may read shipment status but may not modify billing or another tenant's routes. Its credential record needs a stable identifier, tenant ownership, scope, status, creation time, and retirement deadline. The secret value belongs in a secret store; deployment metadata and logs should carry only the identifier. OWASP's Secrets Management Cheat Sheet recommends lifecycle management, least privilege, expiration, revocation, auditing, and automation. Those controls fit this problem better than treating a key as an anonymous string.

**The unit of rotation is the credential record, not the Kubernetes Secret object.** A Secret is one delivery mechanism. The verifier's acceptance set and the workload's active selection determine whether the transition is safe.

The state machine can stay small:

- `pending`: issued but not yet selected by clients
- `active`: preferred for new requests
- `retiring`: still accepted until a deadline, but no longer preferred
- `revoked`: rejected regardless of caches or deployment state

Transitions should be monotonic, except that an operator may abandon a `pending` successor before it becomes active. Do not restore a revoked key during rollback. Roll back the application, then deliver a still-valid credential to it. Reanimating old authority makes incident containment ambiguous.

No resurrection.

## Derive the grace period from the slow path

A grace period is a design input, not a universal number. Start with the maximum expected time for the secret store projection or synchronization path, add the maximum rollout duration, then add the longest legitimate request or queued retry lifetime. Include clock-skew allowance and an operator-response margin. Use observed upper bounds from your own system, then cap abnormal retry behavior rather than allowing it to extend credential life forever.

For example, a team might configure `sync_budget`, `rollout_budget`, `retry_budget`, `clock_skew_budget`, and `operator_margin`. Those names matter more than a magic total because each term has a different owner and alert. If a logistics worker may legally retry a carrier-status lookup for 40 minutes, a 30-minute overlap cannot be justified merely because pods usually roll in 8 minutes. These are illustrative configuration values, not benchmark results.

The control-plane calculation can be expressed without coupling it to a particular store:

```python
from dataclasses import dataclass
from datetime import datetime, timedelta, timezone


@dataclass(frozen=True)
class RotationBudget:
    sync: timedelta
    rollout: timedelta
    retry: timedelta
    clock_skew: timedelta
    operator_margin: timedelta

    def grace(self) -> timedelta:
        return self.sync + self.rollout + self.retry + self.clock_skew + self.operator_margin


def retirement_deadline(budget: RotationBudget) -> datetime:
    return datetime.now(timezone.utc) + budget.grace()
```

This sum is conservative because phases may overlap. That is deliberate for the first rollout. Once telemetry shows which phases overlap reliably, the team can tighten the budget while preserving a documented margin. Cost enters here as a secondary constraint: longer overlap keeps more credentials and audit events live, while very short overlap buys a small reduction in stored state at the expense of a brittle deployment. Storage cost should not drive the security boundary.

There is another limit. If the calculated overlap exceeds the organization's acceptable exposure window for a leaked key, the answer is not to hide the difference in a larger grace value. Reduce retry lifetime, speed up distribution, split the rollout, or use shorter-lived credentials. The security limit wins.

This pattern has a real limitation: it is not suitable when the verifier cannot accept two credential IDs at once, or when clients cannot refresh secrets within the maximum exposure window. In that environment, a brokered short-lived credential or a maintenance window is the more honest design. Dual acceptance also increases temporary secret inventory and audit volume. That trade-off is justified only when uninterrupted traffic matters and revocation remains authoritative.

## Make publication and revocation separate decisions

The rollout protocol needs evidence at each transition. First, create the successor with the same tenant and no broader scope than the predecessor. Add it to the verifier's acceptance set while leaving the predecessor valid. Next, publish a versioned secret payload containing both identifiers and both secret values, with the successor marked preferred. Workloads should use the preferred credential for new calls but retain the retiring credential only for controlled fallback during the overlap.

Do not fall back on every authentication failure. A `403` can mean the credential is valid but lacks scope; retrying with another key can conceal an authorization error. Restrict fallback to the verifier's explicit invalid-or-expired credential outcome, make it one attempt, and record only credential IDs. Never log raw keys.

A generic selector makes the local rule testable:

```python
from dataclasses import dataclass
from datetime import datetime


@dataclass(frozen=True)
class DeliveredKey:
    key_id: str
    secret: str
    preferred: bool
    accept_until: datetime | None


def choose_key(keys: list[DeliveredKey], now: datetime) -> DeliveredKey:
    usable = [key for key in keys if key.accept_until is None or now < key.accept_until]
    preferred = [key for key in usable if key.preferred]
    if len(preferred) != 1:
        raise RuntimeError("expected exactly one preferred credential")
    return preferred[0]
```

The process should load a refreshed payload without exposing it through command arguments or diagnostics. If the application's secret integration requires a restart to observe changes, that restart belongs inside the rollout budget. A readiness check must prove the process has loaded the expected credential version, not make an outbound business request that creates side effects.

Then watch adoption. Count requests by tenant, credential ID, result class, and workload revision, while keeping label cardinality bounded. A useful revocation gate asks whether every intended workload revision acknowledged the new secret version, whether the successor produced successful authenticated traffic, and whether predecessor usage stayed at zero for an observation interval covering expected retries. Deployment completion alone answers none of those questions.

Zero is meaningful only when traffic exists. A quiet tenant can show zero old-key requests and zero new-key requests. For that case, use an authenticated, side-effect-free validation operation if the protocol supplies one, or require explicit workload acknowledgements plus a later tenant-specific check. Do not manufacture a shipment event as a health check. Deliverability systems teach the same lesson: an accepted handoff and an observed end result are separate signals. Credential publication and credential use are separate too.

No traffic, no proof.

After the gate passes, mark the predecessor revoked in the authoritative verifier, remove it from the next delivered payload, and retain its ID and audit history according to policy. If old-key traffic appears after the deadline, reject it and alert on the owning tenant and workload revision. Extending the deadline automatically converts a deployment defect into a silently growing exposure window.

## Test the edge cases before the first tenant

The happy path is uninteresting. Test a replica that starts before publication and becomes ready afterward; a worker that sleeps across the switch; a rollout that pauses halfway; a secret refresh that arrives out of order; and a rollback to code built before dual-key selection existed. Also test revocation during an active security response, where grace is intentionally skipped.

Use fake clocks around retirement logic. Run the same conformance suite against the in-memory verifier and the production adapter: tenant A's key cannot authorize tenant B, a read-only scope cannot mutate a shipment, revoked always beats cached active state, and a repeated rotation request does not create an unbounded chain of valid keys. Keep at most the explicitly modeled active and retiring generations.

The dangerous test shortcut is asserting only that both keys work during overlap. Also assert the negative boundary after it. At the retirement instant, the predecessor must fail even if a client presents a syntactically correct key and even if an application cache still contains its record. The authoritative status has to dominate cache freshness; otherwise revocation latency is whatever the cache timeout happens to be.

Observability deserves a failure test as well. Confirm that traces, structured logs, exception messages, deployment events, and support bundles expose a credential ID or fingerprint rather than the secret. Access to rotation actions and secret reads should be auditable. Alerts should distinguish failed publication, stalled adoption, unexpected old-key use, and rejected revoked-key use because those conditions have different responders.

## A compact tenant-by-tenant rollout

Begin with one low-traffic logistics tenant and inventory every consumer, including scheduled jobs and dormant workers. Add dual acceptance at the verifier before any client prefers the successor. Issue one scoped successor, publish a versioned two-key payload, and roll workloads while measuring acknowledgements and usage by credential ID. Revoke only after the evidence gate and configured deadline agree.

Pause there. Verify that the old ID is rejected, no secret value entered telemetry, and rollback documentation uses the successor rather than reviving the predecessor. Then expand in small tenant batches with a concurrency limit so one control-plane error cannot rotate the entire fleet.

The final decision rule is direct: choose an overlap long enough for the slowest legitimate consumer, but never broader than the accepted exposure window. When those constraints conflict, fix the delivery or retry path before rotating more tenants. This keeps a credential mistake local to one tenant and one scope, which is the property the rollout was meant to preserve.

## Sources

- https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html
