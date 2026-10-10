# Password Reset Email API Explained: Next.js Custom HTML Link Suppression

A Next.js API route for password reset email should leave policy to a backend service: accept every request, disclose nothing about the account, and send the custom HTML link only after recording a current suppression decision. The decisive artifact is not a transport's accepted response. It is a joined trail from request ID to recipient decision, message attempt, and later bounce outcome.

**Short answer:** keep the public API route thin. Normalize the address, create a one-time expiring token only for an eligible account, check the local suppression ledger immediately before dispatch, render a fixed HTML template with an escaped URL, and record the decision. A hard bounce closes the recipient before another recovery message can be attempted. Temporary delivery trouble follows a separate, bounded path.

For a property manager, one login can expose tenant contact details, maintenance records, or owner statements. Recovery has to remain usable without turning invalid addresses into repeated mail or turning audit logs into a second copy of sensitive credentials.

## How should a Next.js API route send a password reset email?

The route returns the same generic response for an unknown address, an active account, and a suppressed recipient. That prevents the response body from becoming an account directory. Rate limits should also be evaluated without revealing which check failed.

Four invariants carry most of the design:

1. A reset token is random, single-use, short-lived, and stored only as a digest. The raw token exists long enough to build the link and is never written to an event or application log.
2. The suppression decision is made against normalized recipient identity at send time, not only when the request first arrives. A bounce can land between those two moments.
3. Template variables are escaped. The recovery URL comes from a configured HTTPS origin plus an encoded token, never from a request-supplied host header.
4. Each transition has an immutable correlation ID and a timestamp. The evidence says why sending was allowed or denied without storing the raw reset link.

These are application invariants. DKIM operates at a different boundary: RFC 6376 defines a domain-level signature that lets a verifier associate a message with a signing domain and detect modification of signed content. It does not decide whether a tenant mailbox is valid, and a valid signature does not override a suppression record. Treat authentication and recipient eligibility as two controls.

No ambiguity here.

The route is an adapter.

A useful failure boundary is the handoff to the mail transport. Before handoff, the application owns eligibility, token state, template correctness, and evidence. After handoff, delivery events may change future eligibility, but an accepted handoff is not proof that the mailbox received the message. Model those states separately.

The concrete trade-off is one extra policy read on the critical path in exchange for evidence tied to the moment of dispatch. If that read cannot complete, recovery email stops even though the account and mail transport may be healthy. That is a real availability cost, and the team should name it in the decision record rather than hide it behind a generic queue retry. For property-management accounts, the choice here is to fail closed because an old invalid address may belong to somebody who no longer has a relationship with the property.

## Decision record: where should suppression live?

The primary decision axis is compliance evidence: can an operator reconstruct what the system knew when it acted? The options differ less in syntax than in the authority they assign to each store.

| Option | Send-time authority | Evidence quality | Failure behavior | Best fit |
|---|---|---|---|---|
| Local suppression ledger | Application database | Joins request, decision, and event history | Fail closed when eligibility cannot be read | Review-heavy recovery flows |
| Transport-only suppression | External transport | Evidence is split across systems and retention rules | Depends on transport state and integration | Low-risk notifications with mature event ingestion |
| Request-time check only | Application snapshot | Explains the early check but not final handoff | A later bounce can race a queued send | Short paths where check and handoff are atomic |

The selected design uses the local ledger as the policy authority and treats transport events as inputs to it. This does not require the application to predict mailbox behavior. It requires the application to preserve the latest known disposition and refuse a send when that disposition is unavailable or terminal.

Stale evidence is no evidence.

A minimal ledger needs fewer fields than teams often expect: a keyed recipient digest, status, reason class, source event ID, first-seen time, last-seen time, and policy version. Keep the clear address in the account system where it already belongs. The evidence table can use a keyed digest so reviewers can correlate events without casually exposing resident addresses. The key and its rotation procedure then become part of the control.

Hard and soft outcomes must not collapse into one Boolean. A permanent invalid-recipient outcome can suppress future recovery email until the address changes or an authorized process clears it. A temporary outcome may permit retry under a bounded policy. An explicit opt-out can have different semantics for security mail than for marketing mail, so encode the message class and applicable policy instead of guessing from one `suppressed` flag.

## Critical path in Python

The framework-specific route should parse the request and call one service. The service below shows the boundary; repository and mailer methods are generic interfaces, and the response is intentionally identical across outcomes.

```python
from dataclasses import dataclass
from hashlib import sha256
from html import escape
from secrets import token_urlsafe
from urllib.parse import quote

GENERIC_RESULT = {
    "ok": True,
    "message": "If the account is eligible, recovery instructions will be sent.",
}


@dataclass(frozen=True)
class ResetRequest:
    email: str
    request_id: str


def normalize_email(value: str) -> str:
    return value.strip().casefold()


def build_html(reset_url: str) -> str:
    safe_url = escape(reset_url, quote=True)
    return (
        "<p>A password reset was requested for your property portal account.</p>"
        f'<p><a href="{safe_url}">Reset your password</a></p>'
        "<p>If you did not request this, you can ignore this email.</p>"
    )


def request_password_reset(req, accounts, suppressions, resets, mailer, audit, config):
    recipient = normalize_email(req.email)
    account = accounts.find_active_by_email(recipient)
    if account is None:
        audit.record(req.request_id, "reset_not_dispatched", {"reason": "ineligible"})
        return GENERIC_RESULT

    decision = suppressions.current_decision(
        recipient, message_class="account_recovery"
    )
    if decision is None or not decision.may_send:
        reason = "eligibility_unavailable" if decision is None else decision.reason_class
        audit.record(req.request_id, "reset_not_dispatched", {"reason": reason})
        return GENERIC_RESULT

    raw_token = token_urlsafe(32)
    token_digest = sha256(raw_token.encode("utf-8")).hexdigest()
    resets.create_single_use(account.id, token_digest, config.reset_expires_at())
    reset_url = (
        f"{config.public_origin}/reset-password?token={quote(raw_token, safe='')}"
    )
    attempt_id = mailer.send_html(
        to=recipient,
        subject="Reset your property portal password",
        html=build_html(reset_url),
        idempotency_key=req.request_id,
    )
    audit.record(
        req.request_id,
        "reset_handed_off",
        {"attempt_id": attempt_id, "policy_version": decision.policy_version},
    )
    return GENERIC_RESULT
```

The omitted adapters still matter. The API layer should reject malformed payloads, apply abuse controls, and generate the request ID. `create_single_use` should invalidate older active reset tokens in the same transaction. The redemption path hashes the presented token, checks expiry and unused state, changes the password, and consumes the token atomically. None of those operations should log the token.

There is a narrow race between the final ledger read and transport handoff. With an outbox worker, store the token record, intent, and audit transition together, then repeat the suppression check immediately before sending. This adds latency and operational machinery, but it closes the gap where a queued job carries an old eligibility decision. For account recovery, that trade is defensible.

## Bounce ingestion and operational checks

Delivery events are untrusted input even when they arrive on a private-looking endpoint. Verify event authenticity according to the transport's documented mechanism, reject duplicates by source event ID, and map external event vocabulary into a small internal taxonomy. Preserve the original reason as restricted evidence when policy requires it; make decisions from the stable internal class.

Ordering is awkward. A delayed temporary event must not overwrite a later permanent invalid-recipient event. Use an explicit precedence rule plus event time, and record ignored transitions. Retries should be idempotent. If ingestion is unavailable, retain the event for replay and stop recovery sends when the local eligibility view is too stale for the declared policy.

This design also has a limitation: a team must operate the ledger, event adapter, replay path, and policy migration. It is not suitable for a low-risk notification system whose owners cannot support that state. In that case, transport-owned suppression can be the more honest choice, provided its retention and outage behavior meet the system's evidence requirement.

Monitor the shape of the pipeline, not message content: reset requests, generic rejections, suppression denials by reason class, handoff attempts, permanent outcomes, temporary outcomes, duplicate events, event age, and ingestion lag. Alert on a sustained loss of events or an abrupt distribution change. Do not put email addresses, HTML bodies, or reset URLs in metric labels.

Test at three levels. Unit tests cover normalization, escaping, status precedence, and the generic response. Contract tests feed representative authenticated events through the adapter, including duplicates and out-of-order delivery. A staging test uses controlled mailboxes and confirms that the visible link points only to the configured origin, the account-enumeration behavior remains constant, and a permanent event blocks the next attempt.

Deliverability basics belong beside these tests. Sign outbound mail with DKIM, keep the visible sender and recovery domain under organizational control, send a text alternative alongside HTML, and avoid mixing security mail with promotional content. Authentication cannot rescue a stale list. Suppression hygiene cannot repair a broken signature. Both need observable ownership.

## Rejected option and its valid use case

The rejected design asks the transport to own suppression completely. It is attractive because there is less local state, and it can be valid for non-sensitive notifications when the team accepts the transport's policy, event retention, lookup behavior, and outage semantics. Document those boundaries before choosing it.

It was rejected here because the property-management application needs one durable explanation for a recovery decision. Splitting request evidence, external suppression state, and bounce history makes that explanation depend on cross-system availability and aligned retention. A local policy ledger also allows a mail transport change without silently changing the meaning of `may_send`.

SMS is a possible recovery channel, not an automatic retry for failed email. It has its own recipient consent, number-reassignment, rate-limit, and delivery-state questions. If the product offers it, model it as a separately enrolled factor with separate evidence; do not copy an email suppression decision into an SMS permission. The same generic public response still applies.

The final acceptance criterion is concrete: an auditor can start with a request ID and establish the policy version, recipient eligibility decision, token lifecycle, handoff attempt, and later disposition without seeing the raw token. If any link in that chain exists only in transient logs or an external dashboard, the architecture is unfinished.

## References

- RFC 6376, DomainKeys Identified Mail (DKIM): https://datatracker.ietf.org/doc/html/rfc6376
- Twilio SMS documentation: https://www.twilio.com/docs/sms
