# How to Debug Inbound Mail with Python (4 Provider Record Checks)

A customer can intend to move inbound support mail while the public DNS still describes two receivers. That mismatch, not the age of either provider, is the constraint that changes the debugging order.

**TL;DR:** Treat the desired mail route as data, fetch every published MX record, normalize names and priorities, and stop the rollout unless the two sets match exactly. A lower MX preference number is tried before a higher one; the higher-numbered record is a fallback, not a harmless historical note. Remove an old provider's MX only after the replacement is verified, then keep reconciling until caches can no longer expose the earlier answer.

This matters in customer support because a partial migration does not look like a clean outage. One sender may reach the new intake while another reaches an old mailbox, so agents see missing tickets rather than an obvious DNS failure. Spam controls and message authentication can add noise, but they should not distract from proving which hosts the domain currently advertises.

Route first. Policy second.

## Why is inbound mail not arriving after provider priorities change?

SMTP senders query the recipient domain's MX records and sort the returned exchanges by preference. The smallest number has the highest preference. If two exchanges share a preference, the sender must randomize them; if the preferred exchange cannot accept mail, a sender can try a less-preferred exchange. RFC 5321 also says a sender must not use an implicit MX fallback when an MX record exists. Those details turn a leftover provider record into an active route, which is the first thing to debug when messages stop arriving.

Suppose `help.example.net` is meant to use only `inbound.support-gateway.example`, but DNS publishes this:

```json
{
  "domain": "help.example.net",
  "intended_mx": [
    {"preference": 10, "exchange": "inbound.support-gateway.example."}
  ],
  "published_mx": [
    {"preference": 10, "exchange": "inbound.support-gateway.example."},
    {"preference": 20, "exchange": "legacy-mail.example."}
  ]
}
```

The record at preference 20 is not disabled. It remains eligible whenever the preference-10 host is unavailable to a particular sending system. An old record at preference 5 is worse: it becomes the first destination. Equal preference creates intermittent splitting by design. Three shapes, three different symptoms.

Do not infer ownership from a control-panel screenshot. Query the authoritative DNS path and compare the complete answer with the declared configuration. Also inspect CNAME indirection at the name being tested: RFC 2181 forbids other data at a CNAME owner, so a customer who tries to combine a CNAME and MX at the same label has described an invalid arrangement rather than a mail-routing strategy.

## Step 1: Turn routing intent into a strict comparison

Store the intended set as `(preference, canonical exchange)` pairs. DNS names are case-insensitive, and presentation commonly includes a trailing dot, so normalize both sides before comparing. Do not compare only the first answer. Do not treat an extra record as a warning. For a migration gate, extras and omissions are both drift.

The following Python accepts resolver output that has already been collected by your DNS library or command runner. Keeping lookup transport outside the comparison makes the decision function deterministic and easy to test.

```python
from dataclasses import dataclass


@dataclass(frozen=True, order=True)
class MX:
    preference: int
    exchange: str


def canonical_name(value: str) -> str:
    return value.strip().lower().rstrip(".") + "."


def normalize(rows: list[dict]) -> set[MX]:
    return {
        MX(int(row["preference"]), canonical_name(row["exchange"]))
        for row in rows
    }


def reconcile(intended: list[dict], published: list[dict]) -> dict:
    wanted = normalize(intended)
    found = normalize(published)
    return {
        "matches": wanted == found,
        "missing": sorted(wanted - found),
        "unexpected": sorted(found - wanted),
    }
```

A set is appropriate because duplicate presentation rows do not create a distinct route. Preference remains part of the identity; changing 10 to 20 changes failover behavior even when the hostname stays constant. That is exactly the sort of tiny edit that broad "hostname exists" checks miss.

I use a hard equality gate for routing because deliverability debugging gets expensive when configuration ambiguity survives deployment. The trade-off is deliberate: a planned backup exchanger must appear in intent, complete with its preference, or the check blocks it. Operators should never have to guess whether an extra target is resilience or residue.

## Step 2: Classify the drift before touching DNS

Once the comparison fails, preserve the raw answer, resolver identity, query time, and authoritative nameserver set in the diagnostic event. A recursive resolver can legitimately show cached data until TTL expiry, while an authoritative answer represents the zone currently being served. Those are different observations, not contradictory evidence.

| Observation | Likely meaning | Action |
|---|---|---|
| Intended pair is missing everywhere | Zone was not updated or the wrong owner name changed | Correct the authoritative zone |
| Unexpected lower-numbered pair exists | Legacy receiver is preferred | Remove it after replacement verification |
| Unexpected higher-numbered pair exists | Legacy receiver remains a fallback | Remove it after replacement verification |
| Unexpected equal-numbered pair exists | Traffic can split between receivers | Stop rollout and remove ambiguity |
| Authority is correct but recursive answers differ | Cached earlier RRset is still visible | Wait and re-query across the TTL window |

"Remove it" does not mean delete first and investigate later. Before changing the RRset, prove that the intended receiver accepts mail for the exact customer domain, maps recipients to the correct support tenant, preserves the original envelope recipient, and rejects unknown recipients with the chosen policy. A TCP connection alone proves almost nothing about ticket creation.

Authentication is a separate track. DMARC, specified in RFC 7489, evaluates message alignment and published policy; it does not select the inbound MX destination. Still, migration owners should inventory TXT records because changing outbound systems or forwarding paths can affect SPF and DKIM alignment. Keep those findings out of the MX verdict so a DMARC issue cannot mask a routing mismatch.

## Step 3: Test the gate with ugly cases

Happy-path tests are too polite. Cover case differences, trailing dots, stale fallbacks, changed priorities, and equal-preference additions. These five tests exercise the comparison boundary without requiring live DNS.

```python
def test_reconcile_mx() -> None:
    intended = [{"preference": 10, "exchange": "INBOUND.example"}]
    equivalent = [{"preference": 10, "exchange": "inbound.example."}]
    assert reconcile(intended, equivalent)["matches"]

    stale_fallback = equivalent + [
        {"preference": 20, "exchange": "old.example."}
    ]
    assert reconcile(intended, stale_fallback)["unexpected"] == [
        MX(20, "old.example.")
    ]

    stale_primary = equivalent + [
        {"preference": 5, "exchange": "old.example."}
    ]
    assert not reconcile(intended, stale_primary)["matches"]

    changed_priority = [
        {"preference": 30, "exchange": "inbound.example"}
    ]
    result = reconcile(intended, changed_priority)
    assert len(result["missing"]) == 1
    assert len(result["unexpected"]) == 1

    equal_peer = equivalent + [
        {"preference": 10, "exchange": "other.example."}
    ]
    assert not reconcile(intended, equal_peer)["matches"]
```

Run the pure tests in continuous integration. Run live reconciliation on domain onboarding, after every requested DNS change, and periodically afterward; the last check catches manual edits and expired migration assumptions. Emit one state-change alert when a domain moves from matching to drifted rather than paging on every poll. Include the missing and unexpected pairs, but avoid logging message content or recipient addresses. Compliance starts with collecting less.

The production worker also needs bounded timeouts and retry jitter around DNS queries. A lookup timeout is `indeterminate`, not `mismatch`. Conflating those states encourages an operator to edit a correct zone during a resolver failure. Represent `match`, `drift`, and `indeterminate` separately, and let only a confirmed match advance automated onboarding.

## Step 4: Roll out without creating a second mystery

Use a compact migration sequence. First, record the current RRset and TTL, configure and verify the new receiver, and lower TTL only if the domain owner agrees far enough ahead for existing cached values to expire. Next, publish the complete intended RRset as one change. Then query the authoritative servers and a small, documented set of recursive resolvers until they converge. Finally, send controlled test messages from independent sending paths and verify that each becomes exactly one support ticket.

Keep the old receiver able to surface late arrivals during the cache window, but do not leave its MX published as an indefinite safety blanket. That creates permanent dual ownership. Define the rollback RRset before the change, give the migration an owner, and preserve timestamps for every observation.

The final acceptance rule is intentionally boring: the authoritative RRset equals intent, observed recursive answers converge after the applicable TTLs, the intended receiver handles the customer domain correctly, and no unexpected exchange remains published. **DNS intent should be machine-checkable, not institutional memory.**

## Sources

References used for the standards behavior in this runbook:

- https://datatracker.ietf.org/doc/html/rfc5321
- https://datatracker.ietf.org/doc/html/rfc2181
- https://datatracker.ietf.org/doc/html/rfc7489
