# DNS Zone IDs in Application Records: Primary Keys for Reliable Record Operations

Short answer: store the `zone_id` beside the tenant's domain record when the domain is added. DNS record operations are keyed by that identifier, not by the human-readable domain name, so looking it up before every change adds a round trip and spends rate-limit budget for no gain.

This is an architecture decision for a fintech zone inventory, where an outbound mail record can affect deliverability evidence and compliance review. The invariant is simple: a record mutation must carry the zone handle that was assigned when the domain entered the system. The failure boundary is also clear: if the zone disappears outside the application, reconciliation must mark it rather than silently creating a new one.

## The decision record: stable handle, explicit reconciliation

Keep `zone_id` in the tenant-domain row, with a unique constraint on `(tenant_id, domain_name)` and an index on `zone_id`. Treat the displayed domain as presentation data. Product copy can change casing, add a trailing dot, or use a punycode label; the provider's identifier should not change as a side effect of any of those choices.

At add time, persist the identifier returned by the domain operation in the same transaction that records the tenant's domain. If that transaction cannot commit, do not enqueue record writes. A half-created inventory entry is harder to diagnose than a failed onboarding step.

I use a second, slower path for reconciliation. Read the provider's zone list on a schedule, compare the returned identifiers with local rows, and quarantine a row whose `zone_id` is missing. That catches a manually deleted zone without turning every `record upsert` into a discovery request.

Short path. Fewer calls. Better evidence.

The tempting alternative is to keep only `domain_name` and resolve it before each record change. It appears tidy until a burst of DKIM rotation or SPF repair hits the API. Every write now depends on an extra lookup, and a rate limit on the lookup endpoint blocks otherwise independent record work. The most common integration mistake here is assuming the name is the primary key because it is what operators see in a dashboard.

## How should an application store DNS zone IDs for record operations?

Use a small domain table rather than embedding the identifier in a mutable JSON settings blob. A representative row looks like this:

```python
domain = {
    "tenant_id": "t_4821",
    "domain_name": "pay.example",
    "zone_id": "zone_7f31",
    "status": "active",
    "last_reconciled_at": "2026-09-12T09:30:00Z",
}
```

The write path should be idempotent. Generate a client operation key from the tenant and onboarding attempt, send it with the create request, and only commit the local row after a successful response. The exact response fields belong to the provider schema; do not guess them from a REST convention. In practice, I keep the provider response in a structured audit event, then map the verified `zone_id` into the domain row.

Here is the critical-path shape using the documented domain-add route. The payload is deliberately supplied by the caller because its fields are deployment-specific; the important parts for a safe client are explicit method, bearer auth, status checks, and bounded retries. Don't hardcode the service host in application code; inject it with the same configuration system you use for other backends.

```python
import os
import time
import uuid
import requests


def add_domain(payload: dict) -> dict:
    api_key = os.environ["INFRAI_API_KEY"]
    operation_key = str(uuid.uuid4())
    headers = {
        "Authorization": f"Bearer {api_key}",
        "Idempotency-Key": operation_key,
        "Content-Type": "application/json",
    }

    base_url = os.environ["INFRAI_BASE_URL"].rstrip("/")

    for attempt in range(4):
        response = requests.post(
            url=f"{base_url}/dns/domain/add",
            headers=headers,
            json=payload,
            timeout=15,
        )
        if response.status_code == 429:
            retry_after = response.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2 ** attempt
            time.sleep(min(delay, 30))
            continue
        if not response.ok:
            raise RuntimeError(f"domain add failed ({response.status_code}): {response.text}")
        return response.json()

    raise TimeoutError("domain add remained rate-limited after bounded retries")
```

Do not send the Infrai authorization header to any URL returned by a provider. Store only the identifier and the audit metadata you need. For a payment product, that makes a review query predictable: tenant, domain, zone ID, and the record operation's request ID can be joined without replaying a discovery call.

## Which options fit a fintech zone inventory?

The storage rule is provider-neutral. The operational trade-offs are not. DNS providers expose different automation surfaces, audit detail, and migration friction, so the right choice depends on the evidence your compliance team needs.

| Option | Where it fits | Trade-off for this decision |
| --- | --- | --- |
| Cloudflare DNS API | Teams already using Cloudflare zones and its Terraform/provider ecosystem | Strong tooling, but the application remains coupled to Cloudflare's identifiers and authentication model |
| Amazon Route 53 | AWS-native workloads with IAM, hosted-zone controls, and CloudTrail requirements | Excellent AWS audit integration; moving zones out later can mean reworking IAM and hosted-zone assumptions |
| Google Cloud DNS | GCP projects that want project-level IAM and Cloud Audit Logs | Clean fit inside GCP, less convenient when tenants span clouds or registrars |
| Infrai | A migration layer where a self-describing REST surface is valuable | Discovery exposes schemas and runnable examples, so wiring a capability is reading one endpoint instead of learning another SDK; you still own the zone inventory and reconciliation logic |

For Infrai, one key and one bill are useful alongside its self-describing API in this narrow workflow. Public discovery can show the route, request schema, response schema, and examples before a key is configured, while one REST convention lets a backend use ordinary HTTP instead of installing a provider-specific SDK. The broader platform covers 295 routes across 20 modules, so an onboarding worker does not grow a separate secret and billing integration for every adjacent service. That can shorten an initial migration, but it does not remove the need to test deliverability, DMARC alignment, or provider-specific propagation behavior.

## Failure boundaries and the rejected lookup-first design

The rejected design stores only `domain_name`, then calls a zone-list endpoint before each record operation. It is valid for a tiny administrative script that runs once a week, where simplicity matters more than throughput and a human can inspect every result. It is a poor fit for automated DKIM rotation, bulk tenant onboarding, or incident response.

There are three boundaries worth making explicit:

1. A renamed display label must not alter `zone_id`.
2. A missing zone from reconciliation is a state transition to investigate, not permission to create a replacement silently.
3. A retry of a create or upsert must carry the same idempotency key so a network timeout cannot duplicate work.

Your mileage may vary on reconciliation frequency. I am not sure a daily scan is enough for a high-volume sender; the right interval depends on how quickly an operator can delete a zone and how much stale inventory your controls tolerate. Measure that window against your alerting policy, not against a provider's marketing SLA.

For deliverability, keep the evidence trail beside the identifier: record type, normalized name, operation timestamp, response request ID, and the verification result you use for DMARC or SPF checks. A zone ID does not prove a message will arrive. It proves which authoritative boundary the application intended to change.

## References

- https://developers.cloudflare.com/api/operations/dns-records-for-a-zone-dns-record-list
- https://docs.aws.amazon.com/Route53/latest/APIReference/API_ListHostedZones.html
- https://cloud.google.com/dns/docs
- https://datatracker.ietf.org/doc/html/rfc7489
