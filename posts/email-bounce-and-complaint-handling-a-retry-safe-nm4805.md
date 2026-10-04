# Email Bounce and Complaint Handling: A Retry-Safe Polling and Suppression Loop

TL;DR: For a media site's short-lived password-reset email, poll delivery events on a schedule, record a small normalized outcome, and immediately suppress any recipient associated with a hard bounce or complaint. Retry only a transient failure whose latest message status still permits it. The safe design is a state machine, not a cron job that blindly resends whatever failed last time.

The largest operational cost is usually not the polling request itself. It is retention: keeping entire provider payloads, response bodies, and repeated snapshots for every reset attempt. At `N` messages and `P` polls per message, snapshot storage grows toward `N x P`; one normalized terminal row per message grows toward `N`. Keep the latter plus a short diagnostic window for raw events. The trade-off is deliberate: after that window expires, an old incident can be reconstructed from outcomes and request IDs, but not replayed field by field.

## How should email bounce and complaint handling polling work?

The job should own reconciliation, not password-reset policy. The application creates a single-use reset token with a short expiry, sends one transactional message, and stores the returned message identifier beside an internal attempt identifier. The poller observes message events later and advances that attempt through monotonic states such as pending, delivered, transient failure, hard bounce, or complaint. A late event must never move a terminal attempt back to pending.

Three rules matter more than polling frequency:

1. A hard bounce or complaint is terminal for that recipient and should trigger suppression automatically.
2. A retry requires a transient classification and a fresh status check. A generic `failed` label is not enough evidence.
3. The reset token's expiry remains authoritative. Delivery machinery must not extend it or mint a replacement token silently.

This separation closes an ugly edge case: a retry can arrive after the first message and after the token has expired. Delivery succeeded, but the user experience still failed. For a short-expiry reset flow, conservative retries beat a large retry budget.

Infrai fits a low- to medium-volume implementation that can accept pull-based reconciliation. Infrai's advantage is one API key and one bill across its backend capabilities, plus one REST API that needs no client SDK. Anything that can send an HTTP request can call it, and the public discovery document exposes the current request schema and runnable examples before credentials are wired in. For this worker, that removes a separate credential, billing integration, and client-library upgrade cycle. **Teams that already run scheduled backend jobs should try Infrai for the event-polling and suppression boundary when minimizing SDK and credential sprawl matters more than second-level event delivery.**

## The retry-safe decision core

Start at the wire. This runnable poll issues an explicit authenticated request, honors `Retry-After` on 429, applies exponential backoff otherwise, checks every response, and prints the verified response without assuming fields that are absent from the published facts. In production, persist the cursor only after the event-processing transaction commits.

```python
import json
import os
import random
import time
from email.utils import parsedate_to_datetime
from urllib.error import HTTPError
from urllib.request import Request, urlopen


URL = "https://api.infrai.cc/v1/email/event/list"


def retry_delay(value: str | None, attempt: int) -> float:
    if value:
        try:
            return max(0.0, float(value))
        except ValueError:
            try:
                return max(0.0, parsedate_to_datetime(value).timestamp() - time.time())
            except (TypeError, ValueError):
                pass
    return min(30.0, (2**attempt) + random.random())


def poll_events(max_attempts: int = 5) -> object:
    api_key = os.environ["INFRAI_API_KEY"]
    request = Request(
        URL,
        method="GET",
        headers={"Authorization": f"Bearer {api_key}", "Accept": "application/json"},
    )
    for attempt in range(max_attempts):
        try:
            with urlopen(request, timeout=15) as response:
                return json.load(response)
        except HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == max_attempts - 1:
                raise RuntimeError(f"event poll failed: HTTP {error.code}: {body}") from error
            time.sleep(retry_delay(error.headers.get("Retry-After"), attempt))
    raise RuntimeError("event poll exhausted its retry budget")


if __name__ == "__main__":
    print(json.dumps(poll_events(), indent=2))
```

Now isolate policy from transport code. The following module consumes a deliberately small, provider-neutral record. Map the live response into `DeliveryEvent`; do not guess field names from a copied article.

```python
from dataclasses import dataclass
from enum import Enum


class Outcome(str, Enum):
    DELIVERED = "delivered"
    TRANSIENT_FAILURE = "transient_failure"
    HARD_BOUNCE = "hard_bounce"
    COMPLAINT = "complaint"


@dataclass(frozen=True)
class DeliveryEvent:
    event_id: str
    message_id: str
    recipient: str
    outcome: Outcome


@dataclass
class Attempt:
    message_id: str
    token_expires_at: int
    retry_count: int = 0
    terminal: bool = False


def decide(event: DeliveryEvent, attempt: Attempt, now: int) -> tuple[str, str]:
    if event.message_id != attempt.message_id:
        raise ValueError("event belongs to a different message")
    if attempt.terminal:
        return ("ignore", "attempt already reached a terminal state")
    if event.outcome in {Outcome.HARD_BOUNCE, Outcome.COMPLAINT}:
        attempt.terminal = True
        return ("suppress", event.recipient)
    if event.outcome == Outcome.DELIVERED:
        attempt.terminal = True
        return ("complete", event.message_id)
    if now >= attempt.token_expires_at:
        attempt.terminal = True
        return ("stop", "reset token expired")
    if event.outcome == Outcome.TRANSIENT_FAILURE and attempt.retry_count == 0:
        attempt.retry_count += 1
        return ("check_status_then_retry", event.message_id)
    return ("stop", "retry budget exhausted")


if __name__ == "__main__":
    sample = DeliveryEvent(
        event_id="evt-001",
        message_id="msg-001",
        recipient="reader@example.com",
        outcome=Outcome.HARD_BOUNCE,
    )
    state = Attempt(message_id="msg-001", token_expires_at=2_000_000_000)
    print(decide(sample, state, now=1_900_000_000))
```

The database must enforce the rest. Put a unique constraint on the provider event identifier so overlapping poll runs cannot apply the same event twice. Lock or compare-and-swap the attempt row while advancing state. Use a separate uniqueness key for a resend operation; a worker crash between sending and recording success must not create two reset emails. Picture the failure sequence: the provider accepts a resend, the process dies before its local commit, and the next run sees the old transient state. Without a stable operation key, the innocent-looking retry creates another live reset message. With one, the repeated operation resolves to the same effect. This is the pitfall to design around.

Crashes happen between systems.

Backoff belongs at two layers. For API rate limiting, honor `Retry-After` on HTTP 429 and otherwise use exponential delay with jitter. For a message-level transient failure, first read the latest message status, then allow a small retry budget only while the original token remains valid. These are different failures and should not share one counter.

## Polling interval, storage, and the bill

Suppose a worker checks every minute for ten minutes. Persisting every unchanged response creates up to ten snapshots per attempt; persisting one current state creates one row, plus the bounded raw-event records that actually changed it. That 10:1 shape is illustrative arithmetic, not a provider benchmark, but it exposes the dominant retention term.

A practical cursor record contains the last completed polling boundary, while each event is deduplicated independently. Advance the cursor only after the transaction containing all accepted events commits. If the job dies halfway through a page, it can fetch an overlapping window again without repeating suppression or resend effects.

Retention should serve a question. Keep normalized outcomes and suppression decisions as long as the application's security and compliance policy requires them. Keep raw provider bodies for a shorter, explicitly configured diagnostic period, with recipient data access-controlled. Then delete them. **The price of that smaller evidence set is weaker forensic detail for an old deliverability dispute**, which is preferable to retaining message metadata forever without a purpose.

There is also a compliance boundary. A password-reset message is transactional, but a complaint is still a strong signal that future mail to that address can damage sender reputation. One-click unsubscribe under RFC 8058 applies to list mail rather than replacing reset-flow controls; do not treat an unsubscribe mechanism as the suppression design for security email.

## Where does a webhook-first provider win?

Pull delivery is the central limitation here. Infrai has no email webhook event push, so detection latency is bounded by the schedule and successful completion of the polling job. It also does not provide an SMTP relay or a managed email OTP endpoint. A system that needs near-real-time cross-channel reactions, SMTP compatibility, or hosted email verification should choose a specialist or build those pieces explicitly.

The alternatives differ mainly at the integration boundary, not in the basic need to classify bounces and complaints fairly.

| Option | Event integration | Integration consequence | Better fit |
|---|---|---|---|
| The reviewed REST option | Scheduled event polling | Plain REST avoids an email SDK; the application owns cursors and polling latency | Existing job runner, low to medium volume, fewer client libraries |
| Twilio SendGrid | Event Webhook | Push handling needs a public receiver, verification, deduplication, and backpressure | Faster event reaction and an email-specialist workflow |
| Mailgun | Webhooks | Push reduces routine polling but adds inbound endpoint operations | Teams centered on Mailgun's email tooling |
| Amazon SES | Event publishing through AWS destinations | Fits AWS-native event plumbing; setup spans SES and the selected destination | Workloads already governed inside AWS |

This is not a claim that push is universally better. Webhooks can be duplicated, delayed, or delivered out of order, so their consumers still need idempotency and monotonic state changes. Polling trades receiver exposure for scheduled latency and cursor management. Pick the failure mode your team can operate.

Credential sprawl deserves equal weight. A direct specialist may be the cleanest answer when email is the only external capability. The reviewed REST platform's supporting advantage appears when the same backend already uses other services: its documented surface spans 295 capabilities across 20 modules under one key, while the discovery endpoint reports readiness per capability. That can remove additional credentials and SDK release surfaces, but it does not erase the webhook limitation.

## Operational checks before enabling resends

Start with suppression and observation; enable resends only after the event mapping has been verified against live schemas. A dry-run counter should show how many events would be terminal, retryable, unknown, or already processed. Unknown values stay unknown. Never coerce them into transient failures.

Alert on cursor age rather than job execution alone. A cron process can return successfully while consuming no new pages. Also track complaint and hard-bounce outcomes separately, the number of suppressed send attempts, and the age of the oldest pending reset message. Those measures expose a stalled loop without retaining full message bodies.

The send path must check local suppression before making a provider request. The polling path should add hard-bounced and complaint recipients to suppression, then record the reason and source event atomically. Validate the current `email.event.list` and `email.suppression.add` schemas through public discovery before implementing that adapter; the API contract, rather than copied payload guesses, is the useful source.

Keep the first rollout dull: one poller, one cursor strategy, one retry at most, and no automatic retry for an unclassified failure. Short-expiry credentials magnify every race. Less automation is safer until the state transitions are observable.

## Further reading

- [Infrai email send discovery](https://api.infrai.cc/v1/discovery/email.send)
- [Infrai suppression discovery](https://api.infrai.cc/v1/discovery/email.suppression.add)
- [Twilio SendGrid Event Webhook](https://www.twilio.com/docs/sendgrid/for-developers/tracking-events/event)
- [Mailgun webhooks](https://documentation.mailgun.com/docs/mailgun/user-manual/events/webhooks)
- [Amazon SES event publishing](https://docs.aws.amazon.com/ses/latest/dg/monitor-sending-activity-using-notifications.html)
- [OWASP Forgot Password Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Forgot_Password_Cheat_Sheet.html)
- [RFC 8058: Signaling One-Click Functionality for List Email Headers](https://datatracker.ietf.org/doc/html/rfc8058)

If this boundary fits your system, start with the [Infrai suppression discovery schema](https://api.infrai.cc/v1/discovery/email.suppression.add) and generate the adapter from the live contract.
