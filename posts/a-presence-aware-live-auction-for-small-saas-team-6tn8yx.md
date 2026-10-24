# A Presence-Aware Live Auction for Small SaaS Teams — Realtime API Bidding Order

TL;DR: Accept bids through one authoritative command path, assign order when the application commits them, and distribute the resulting event log to clients. Treat presence as a short-lived, approximate lease rather than proof that a bidder can receive the next update. For a small SaaS team running a live e-commerce auction, this split is more important than the choice between WebSocket, Server-Sent Events, or a data channel.

The deciding constraint is fairness. A browser timestamp reports what one device believed; it cannot establish a global order across devices with different clocks and network paths. The server therefore owns admission, validation, sequencing, and the close decision. Realtime delivery keeps the room responsive, but it does not get to rewrite the auction record.

This is the same discipline I bring to OTP delivery: a green connection indicator is not evidence that the user received the decisive message. Presence describes recent contact. Durable state answers what happened.

## How Should a Small SaaS Team Approach a Live Auction Bidding API?

Write the invariants before choosing a transport. The auction accepts an offer only while the lot is open, only when it exceeds the current valid amount under the auction's increment rule, and only once for a given idempotency key. Every accepted offer receives a monotonically increasing server sequence. The close operation and offers compete in that same serialized domain, so there is an unambiguous winner at the boundary.

Ordering is policy.

**Arrival at the authoritative commit point defines order.** That is a policy choice, not a claim that the internet delivers packets fairly. Publish it in the auction rules. A bidder with a slower route may submit earlier by wall-clock time and still commit later. Client clocks must never break the tie.

The most dangerous boundary is between accepting a write and broadcasting it. If the process commits offer 1842 and dies before sending the message, reconnecting clients still need to discover 1842. An outbox in the same transaction as the auction mutation, or a durable log written by that transaction, closes that gap. The broadcaster may run twice, so consumers deduplicate by auction ID plus sequence.

Gaps matter.

Closing is another command, not a timer callback with special authority. A scheduler can request a close after the advertised deadline, but the serialized state transition decides whether a competing bid committed first. Keep the legal or business definition of the deadline separate from implementation details, and record enough server-side timing data to audit the decision.

## Decision Record and Failure Boundaries

The decision is a server-authoritative write API paired with a replayable event stream. A single writer per auction is conceptually clean; a database transaction that locks or conditionally updates one auction row can provide the same serialization boundary for a modest service. Shard by auction ID only when measured contention justifies it.

| Approach | Ordering authority | Recovery after a missed event | Presence implications | Decision |
|---|---|---|---|---|
| Request/response writes plus replayable push | Commit sequence | Resume from the last confirmed sequence | Separate lease and heartbeat | Adopt |
| Peer data channels for bid submission | Peer or added coordinator | Must be designed separately | Peer connectivity can be mistaken for eligibility | Reject for authoritative bids |
| Polling snapshots only | Server state | Fetches current state but may hide intermediate order | Presence requires another signal | Valid for low-interaction sales |

WebRTC exposes an `RTCDataChannel`, including ordered delivery configuration, but ordered transport between endpoints does not create an auction-wide commit order. It can still be useful for non-authoritative room features, such as a live poll or transient reaction stream, where a missed intermediate event can be reconciled from a snapshot. Keep bids on the authoritative path.

Presence has its own invariant: online means the service observed a valid session signal within a configured lease window. It does not mean the tab is visible, the bidder saw the latest amount, or the next packet will arrive. Count one user once even if that user has three tabs. Store connection IDs under a user session, expire them independently, and derive the displayed participant count from users with at least one live lease.

Presence can lie.

Short disconnects create noisy join and leave events. A small grace interval can reduce flicker, but it delays departure detection. Pick that interval from the product's tolerance for stale attendance, then measure it. For an auction, bidder controls should depend on an authenticated command response and current auction state, not on the presence badge. For an in-session poll, the denominator must also be explicit: invited users, users present at poll open, or users present at close produce different results.

## The Critical Path

The following Python models the transaction boundary; the same state machine belongs behind a Node.js handler. `store.transact` must serialize changes for one auction and commit the outbox record atomically with the new state. The example deliberately leaves authentication and money representation behind typed domain interfaces, because improvising either inside a socket callback is where correctness leaks.

```python
from dataclasses import dataclass
from datetime import datetime
from decimal import Decimal


@dataclass(frozen=True)
class BidCommand:
    auction_id: str
    bidder_id: str
    amount: Decimal
    idempotency_key: str


def place_bid(command: BidCommand, now: datetime, store):
    def decide(tx):
        prior = tx.find_result(command.auction_id, command.idempotency_key)
        if prior is not None:
            return prior

        auction = tx.lock_auction(command.auction_id)
        if auction.status != "open" or now >= auction.closes_at:
            return tx.record_rejection(command, "auction_closed")

        minimum = auction.current_amount + auction.increment
        if command.amount < minimum:
            return tx.record_rejection(command, "amount_too_low")

        sequence = auction.last_sequence + 1
        event = {
            "type": "bid.accepted",
            "auction_id": command.auction_id,
            "bidder_id": command.bidder_id,
            "amount": str(command.amount),
            "sequence": sequence,
            "committed_at": now.isoformat(),
        }
        tx.update_auction(auction, command.amount, sequence)
        tx.append_outbox(event)
        return tx.record_acceptance(command, event)

    return store.transact(command.auction_id, decide)
```

The idempotency lookup must return the original outcome, including a rejection, rather than evaluate a retry against newer state. Otherwise a client that times out can receive a different answer for the same logical attempt. Keep the key scoped to the authenticated bidder and auction, enforce uniqueness at storage level, and avoid logging credentials or sensitive session tokens.

Retries are normal.

On reconnect, the client sends its last applied sequence. The service replays later events or returns a fresh snapshot carrying the latest sequence; after installing that snapshot, the client applies only higher sequences. A gap triggers recovery. It must not be papered over by rendering event 1844 immediately after 1842.

Slow consumers need a firm boundary. Cap each connection's send buffer, stop adding messages when it crosses the limit, and require snapshot recovery after disconnect. This trades a brief resynchronization for bounded memory and protects the room from one stalled tab. Observe commit latency, rejected-command reasons, sequence-gap recoveries, outbox age, reconnect rate, and presence-lease churn. Those signals distinguish write-path trouble from delivery trouble without treating every reconnect as an auction failure.

This approach has real limitations. Serializing each auction creates a hot coordination point, durable replay adds storage and operational work, and conservative backpressure can disconnect a legitimate bidder on a poor connection. It is a poor fit for a casual poll whose intermediate votes have no business consequence; a periodic snapshot may be easier there. It is also insufficient by itself for multi-region active writes, where the team must define a single home region per auction or adopt a consensus-backed ordering design and accept the added latency and operational burden. The trade-off is deliberate: simpler, auditable order for one auction in exchange for avoiding unconstrained writes everywhere.

## Deployment, Tests, and the Rejected Path

Test concurrency at the state-machine boundary. Submit two valid offers against the same starting amount and assert that the committed sequence is unique, ordered, and consistent with the final amount. Race a close command against a bid. Retry the same idempotency key before and after later offers. Kill the broadcaster after commit and verify that the outbox delivers the event after restart. Then force a sequence gap and check that the client pauses incremental rendering until recovery finishes.

Use one illustrative load shape tied to the product, not a generic vanity benchmark. For example, replay the expected number of watchers while concentrating bid commands near close, then report queue age and commit latency separately from fan-out latency. The exact thresholds belong in the service-level objective; invented universal targets would be misleading.

I learned the distinction from OTP delivery gaps: a provider accepting a message and a person receiving it are separate events. Auction transport deserves the same skepticism. An accepted command needs a durable result, while a pushed update is merely one delivery attempt.

Deploy the schema and readers before writers emit a new event version. Consumers should ignore additive fields they do not understand, while incompatible semantic changes deserve a new event type. Redact bidder identifiers from general telemetry, define retention for the audit record, and involve counsel in auction rules and regional obligations. Fair sequencing is an engineering property; legal fairness is broader.

The rejected design sends authoritative offers over peer data channels. Its attraction is obvious: the room already has low-latency peer connectivity. Its failure boundary is wrong for this job, because the application still needs a coordinator to authenticate bidders, enforce increments, settle concurrent offers, persist the result, and close the lot. Ordered delivery on one channel does not settle order across several bidders.

There is a valid use case. Data channels can carry ephemeral reactions, cursor-like activity, or a live poll whose accepted result is separately recorded by the server. If those messages disappear, the auction ledger remains intact. Poll results that trigger inventory, payment, or contractual action should move onto the same authoritative command pattern as bids.

**Choose the write boundary first.** A small team can change its delivery transport later if durable events, resume sequences, and presence leases are explicit. It cannot cheaply reconstruct a defensible order after client timestamps and best-effort broadcasts have already become the record.

## References

- W3C, WebRTC 1.0: Real-Time Communication Between Browsers: https://www.w3.org/TR/webrtc/
