# Node.js Password Reset Email Retries Without Duplicate Links or Suppression Confusion

TL;DR: Treat a password-reset request, a reset credential, and an email delivery attempt as three different records. A retry may create another delivery attempt, but it must not silently mint another credential or erase the reason a bounced or suppressed recipient was skipped. For a B2B marketplace, that separation gives support a direct answer when a seller cannot enter the dashboard to handle a new order: what the application authorized, what the mail pipeline attempted, and what evidence justified the final delivery decision.

The tempting repair is to retry the whole password-reset job. That mixes identity security with transport recovery. It can produce two messages with different links, while the queue still reports one logical request. It can also keep pushing a known-bad address back toward the transport. The safer boundary is narrower: retry delivery from a stored message intent, and rotate a credential only through an explicit security decision.

## How can password reset email retries cause duplicate links?

Start with identifiers, not inbox screenshots. Assign one `recovery_request_id` when the seller asks for help, one `credential_id` when the application authorizes a reset credential, and a new `attempt_id` for every handoff to the email transport. Put the first two identifiers in the message metadata and logs. Keep the credential itself out of logs.

This makes the main diagnostic question mechanical. If two received messages have the same `credential_id`, delivery was duplicated. If they have different credential identifiers under one recovery request, application or worker logic issued twice. If the identifiers differ at the request level, the system accepted two requests, and the investigation moves to request admission rather than retry behavior.

The distinction matters more than the queue's retry count. A worker can crash after handing off a message but before recording success. On its next run, it cannot infer from the missing local acknowledgment that no handoff occurred. Design for that ambiguity. Persist the intended message before sending, give every attempt its own state transition, and use a stable idempotency key at any boundary that supports one. Where a downstream boundary cannot deduplicate, the ledger still exposes uncertainty instead of pretending that an unacknowledged attempt was never sent.

Short records beat clever logs.

Consider a synthetic trace with three IDs and four timestamps. At 09:00:00, request `R7` creates credential `C4` and intent `M2`. At 09:00:02, attempt `A1` reaches the handoff call, but the caller times out and records `unknown`. At 09:00:07, `A2` reuses `M2` and `C4`; the inbox later displays two messages carrying the same credential. That is a duplicate delivery under uncertainty. Change only one fact: if `A2` carries credential `C5`, the retry path crossed the issuance boundary and created a second authority. Change another: if `A2` is recorded as `suppressed` before handoff because a bounce observation arrived at 09:00:05, then only `A1` could have left the application. This trace is not a production claim or timing target. It is a fixture that forces operators and tests to distinguish three failure classes that look identical in a seller's screenshot. The identifiers do the work; subject lines do not.

For this marketplace flow, the message intent can include `purpose="seller_password_recovery"` and the order-independent reason `access_needed_for_new_order`. Do not put order details into the recovery URL. The notification and the authentication credential have different retention, disclosure, and authorization concerns, even when the same business event prompted them.

## Preserve the decision before contacting the transport

A compliance evidence trail should explain a decision without retaining the reset secret. Store a keyed digest of the credential, its issuance and expiry times, the subject account, and its lifecycle state. Store message rendering inputs separately, with an approved template version. Then store each delivery attempt with the suppression decision observed at send time, the transport correlation value if one was returned, and the outcome category.

Use a small vocabulary that your application owns. For example, `queued`, `handed_off`, `deferred`, `rejected`, and `unknown` describe attempts without assuming that every transport uses the same event names. `suppressed` should be a decision recorded before handoff, not a synonym for every failed delivery. A hard bounce observation may cause a later suppression decision under your policy, but those are two events and should retain two timestamps.

The following Python sketch shows the boundary. The example values are configuration choices, not universal limits. Its important property is that retrying `deliver()` does not call `issue_credential()`.

```python
from dataclasses import dataclass
from datetime import datetime, timezone
from enum import Enum
from typing import Protocol
from uuid import UUID, uuid4


class AttemptState(str, Enum):
    SUPPRESSED = "suppressed"
    HANDED_OFF = "handed_off"
    UNKNOWN = "unknown"


@dataclass(frozen=True)
class MessageIntent:
    recovery_request_id: UUID
    credential_id: UUID
    recipient_id: UUID
    template_version: str
    purpose: str


class EvidenceStore(Protocol):
    def suppression_reason(self, recipient_id: UUID) -> str | None: ...
    def begin_attempt(self, intent: MessageIntent, attempt_id: UUID) -> None: ...
    def finish_attempt(
        self,
        attempt_id: UUID,
        state: AttemptState,
        detail: str,
        observed_at: datetime,
    ) -> None: ...


class MailTransport(Protocol):
    def hand_off(self, intent: MessageIntent, idempotency_key: str) -> str: ...


def deliver(
    intent: MessageIntent,
    evidence: EvidenceStore,
    transport: MailTransport,
) -> UUID:
    attempt_id = uuid4()
    evidence.begin_attempt(intent, attempt_id)

    reason = evidence.suppression_reason(intent.recipient_id)
    if reason is not None:
        evidence.finish_attempt(
            attempt_id,
            AttemptState.SUPPRESSED,
            reason,
            datetime.now(timezone.utc),
        )
        return attempt_id

    key = f"recovery-delivery:{intent.recovery_request_id}:{intent.credential_id}"
    try:
        correlation_id = transport.hand_off(intent, idempotency_key=key)
    except TimeoutError:
        evidence.finish_attempt(
            attempt_id,
            AttemptState.UNKNOWN,
            "handoff_timeout",
            datetime.now(timezone.utc),
        )
        raise

    evidence.finish_attempt(
        attempt_id,
        AttemptState.HANDED_OFF,
        correlation_id,
        datetime.now(timezone.utc),
    )
    return attempt_id
```

The timeout branch is deliberately uncomfortable. “Unknown” is honest: the transport might have accepted the handoff before the caller timed out. Automatically generating a fresh link at that point hides the ambiguity and broadens the security event. A retry policy should first reuse the existing intent while it remains valid, then stop at a documented limit or expiration. Credential rotation belongs to a separate command with its own reason and actor.

Template rendering needs the same discipline. Pin the template version on the intent and render from the same stored inputs on every attempt. Mustache variables are escaped by default, while triple braces or ampersands produce unescaped output; review that distinction rather than letting a retry select whatever template happens to be current. Sections also depend on whether a value is false or an empty list. A missing seller name should therefore be covered by a fixture, not discovered in a recovery email.

## Read suppression and bounce evidence in order

Debugging begins with the application ledger, then moves outward. First locate the recovery request and count credential records. Next list its delivery attempts in timestamp order. Only after that should support correlate a handoff with transport events and the recipient's current suppression state. This order prevents a current suppression entry from being mistaken for the reason an earlier attempt failed.

| Evidence | Question it answers | Misreading to avoid |
|---|---|---|
| Recovery request | Did the application admit one request or several? | Treating repeated UI submissions as a queue retry |
| Credential lifecycle | Was a second reset authority issued? | Calling two distinct links a mail duplicate |
| Message intent and template version | Was the same content authorized for retry? | Rendering a newer template during replay |
| Delivery attempt | Was handoff skipped, accepted, rejected, or left unknown? | Converting a timeout into a definite failure |
| Bounce observation | What delivery feedback arrived, and when? | Assuming it existed before the attempted send |
| Suppression decision | Why was a later handoff blocked? | Treating suppression as proof that no earlier message left |

For a bounced seller address, record the original observation separately from the policy action derived from it. Remediation might require correcting the account address, confirming ownership through an approved channel, or waiting for an authorized review. The email worker should not quietly remove a suppression entry. That would destroy the explanation for why one attempt was skipped and another was allowed.

Do not expose this detail to an unauthenticated reset requester. A public response that reveals whether an account exists, whether its mailbox bounced, or whether it is suppressed turns operational evidence into account-discovery data. Support tooling can show more, but access to it should be authorized and audited. The seller-facing message can remain generic while the internal case links the request, credential, attempts, and policy decisions.

## Make diagnosis reproducible without replaying production mail

The useful test matrix is small but sharp. Exercise a clean handoff, a known suppression, a timeout after possible acceptance, two concurrent workers claiming the same intent, and a template input with an absent optional field. Assert counts as well as states: one recovery request, one credential, and the expected number of attempts. Also assert that logs contain identifiers but no raw token or complete recovery URL.

Concurrency deserves a real database constraint. A process-local lock cannot protect two workers. Claim the intent transactionally, and make the durable idempotency key unique for the logical delivery. The exact schema depends on the database, but the invariant does not: a worker may append attempt evidence; it may not append an extra credential as a side effect of transport retry.

There is a real trade-off. This ledger adds schema migrations, retention policy, access control, and reconciliation work; it is not appropriate for a prototype with no account recovery flow, and it will not prove inbox placement because application evidence stops at the boundaries it can observe. A team must choose retention periods from its own legal and operational requirements, not copy an example. The design is useful when the cost of an unexplained credential or delivery decision exceeds that ongoing operational burden.

For diagnostic automation, expose a narrow, typed operation such as “inspect recovery evidence by request ID,” rather than giving an agent broad access to message bodies or arbitrary account searches. Tool definitions work best when their names and descriptions are unambiguous and their input schemas constrain the allowed arguments. Return redacted, structured fields. This makes an automated summary reviewable against the same ledger a human operator sees.

Observability should count transitions, not addresses. Track the ratio of unknown handoffs, suppression decisions by reason category, credential issuances per recovery request, and intents that remain unfinished past their deadline. Alert on broken invariants, especially more than one active credential for one request. Aggregate metrics can reveal a delivery gap without placing recipient data in metric labels.

## Roll out the ledger without changing the recovery contract

Begin by assigning identifiers and writing evidence alongside the existing path. For one release window, compare the new ledger with existing queue and transport records without using it to block sends. Then enforce the single-credential invariant, move suppression checks ahead of handoff, and finally route retries through the stored message intent. Each step is reversible at the control-flow level while preserving the evidence already written.

Keep the acceptance criterion plain: support can explain a seller's recovery outcome from stored identifiers and timestamps, without sending another email and without reading the secret link. Once that is true, duplicate messages, duplicate credentials, bounces, and suppression decisions stop collapsing into the same vague “email retry” incident.

## Sources

References:

- https://mustache.github.io/mustache.5.html
- https://platform.claude.com/docs/en/agents-and-tools/tool-use/overview
