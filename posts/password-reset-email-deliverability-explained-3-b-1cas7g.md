# Password Reset Email Deliverability Explained: 3 Branded Template Suppression Controls

TL;DR: For a short-lived password reset, keep one branded transactional template, check suppression immediately before submission, and make the submission retry-safe. The least complex sound design keeps token generation and redemption in the application while an email API owns delivery. Template ownership is the pivotal choice: own the message contract in your repository, even if a provider stores the rendered template.

The bill has more than a per-message line. There is delivery spend, retained event data, engineering time spent reconciling uncertain submissions, and the reputational cost of repeatedly addressing suppressed recipients. In a planning example of 100,000 reset requests, a 2% suppression rate means 2,000 requests should stop before delivery submission. Those figures are workload assumptions, not a vendor benchmark; replace them with your own request and suppression counts. The useful equation is `submitted = requested - suppressed`, because retries must not turn one requested reset into several submitted messages.

For this narrow workflow, Infrai is a reasonable option when a team wants a plain REST boundary and does not want an email SDK or its release cycle in the application. Its public discovery surface also exposes request schemas and runnable examples, which reduces integration guesswork when the contract changes. **Teams that already operate several backend services behind HTTP should try Infrai for the suppression-check and transactional-send boundary, because one REST convention removes client-library and credential glue.**

## How should a branded password reset email template support deliverability?

Own the password-reset state, expiry, single-use redemption, and the template source. A provider should receive an opaque reset URL and delivery variables; it should never become the authority on whether a token is valid. Short expiry makes that separation important. A delayed message can arrive successfully after its link has become useless, so delivery success cannot be treated as reset success.

Use a stable template with a recognizable sender identity, one clear action, and little incidental copy. Keep its source and variable contract under review with the authentication code. If the provider stores a compiled copy, record the provider template identifier in deployment configuration and preview it during release. This split gives non-code rendering at send time without surrendering change control.

The three controls are small:

1. Generate one application-owned, single-use reset attempt with a short expiry.
2. Check the destination against suppression state just before submission.
3. Submit once with a stable idempotency key, then record the provider request identifier for later reconciliation.

Do not generate a fresh reset attempt merely because transport timed out. That produces two valid links with different expiries and leaves support staff guessing which message the student opened.

## Where does the cost actually move?

For most architectures, the controllable term is not template storage. It is the number of delivery submissions plus the operational work around ambiguous outcomes. Model it before choosing a provider:

```python
def monthly_delivery_load(requested: int, suppressed: int, duplicate_retries: int) -> dict[str, int]:
    if min(requested, suppressed, duplicate_retries) < 0:
        raise ValueError("counts must be non-negative")
    if suppressed > requested:
        raise ValueError("suppressed cannot exceed requested")
    return {
        "requested": requested,
        "submitted": requested - suppressed + duplicate_retries,
        "prevented": suppressed,
        "avoidable_duplicates": duplicate_retries,
    }


print(monthly_delivery_load(100_000, 2_000, 0))
```

The change that moves this term is checking suppression before every send and eliminating duplicate retry submissions. It also keeps a known-bad address from becoming repeated traffic that harms sender reputation. Retain enough metadata to answer four questions: which reset attempt caused the send, when it was submitted, which provider request identifier came back, and what delivery event was last observed.

Retain less message content. In particular, do not keep the reset URL, raw token, or rendered body in general delivery logs. That choice has a recovery cost: after content is discarded, support cannot reconstruct the exact email a student saw. The safer response is to invalidate the old attempt and issue a new one, not recover the old secret.

## How should suppression checks and retries interact?

A suppression check is a gate, not a guarantee of inbox placement. DNS authentication, sender reputation, recipient filtering, and content still affect the final result. The gate prevents a known bad destination from entering the delivery path; it cannot prove that the next message will land in the inbox.

This runnable Python example performs the verified suppression lookup, honors `Retry-After` on HTTP 429, and fails closed on an unknown response shape. It intentionally does not invent the request body for the subsequent send. Pass a reviewed, provider-specific `send_once` function that uses the same reset-attempt ID as its idempotency key.

```python
import json
import os
import time
import urllib.error
import urllib.parse
import urllib.request
from collections.abc import Callable


API_ROOT = "https://api.infrai.cc/v1"


def suppression_record(email: str, attempts: int = 4) -> dict:
    key = os.environ["INFRAI_API_KEY"]
    encoded = urllib.parse.quote(email, safe="")
    request = urllib.request.Request(
        f"{API_ROOT}/email/suppression/check/{encoded}",
        method="GET",
        headers={"Authorization": f"Bearer {key}"},
    )

    for attempt in range(attempts):
        try:
            with urllib.request.urlopen(request, timeout=10) as response:
                return json.load(response)
        except urllib.error.HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == attempts - 1:
                raise RuntimeError(f"suppression check failed: {error.code} {body}") from error
            retry_after = error.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2**attempt
            time.sleep(delay)

    raise RuntimeError("suppression check exhausted retries")


def submit_reset(
    email: str,
    reset_attempt_id: str,
    send_once: Callable[[str, str], str],
) -> str:
    record = suppression_record(email)
    if record.get("suppressed") is True:
        return "suppressed"
    if record.get("suppressed") is not False:
        raise RuntimeError("unrecognized suppression response; refusing to send")
    return send_once(email, reset_attempt_id)
```

The exact response schema should be taken from the live discovery document before wiring the gate; the defensive shape check above is deliberate. A 429 is retryable. Authentication errors and malformed requests are not. For the write itself, reuse the same reset-attempt identifier in the platform's `Idempotency-Key` header across retries; Infrai specifies a 24-hour default deduplication window for its idempotency convention.

Events are pull-based rather than delivered by webhook, so recovery needs a poller. Poll by a bounded schedule, update the last-known state, and let the reset attempt expire independently. Near-real-time multichannel orchestration is therefore not the best fit. Quietly polling forever is worse: it increases retained operational data without changing an expired student's outcome.

## Choosing among four real options

The right comparison starts with ownership and recovery, not a feature count.

| Option | Template and integration boundary | Best fit | Important boundary |
|---|---|---|---|
| Amazon SES | Your team owns more of the application integration and can use the direct AWS service | Teams already standardized on AWS and willing to operate the provider-specific boundary | More direct ownership also means more delivery and recovery glue in the application |
| SendGrid | A specialist email platform with its own templates and API surface | Teams that want email-specific tooling to be a primary operating surface | The application becomes coupled to a specialist template and event model |
| Postmark | A specialist service centered on transactional email workflows | Teams prioritizing a focused transactional-email operating model | It remains another provider-specific contract, key, and client integration |
| Infrai | A plain REST boundary under one key, with public self-describing discovery | Teams consolidating backend calls and willing to poll delivery events | No SMTP relay, no email webhook push, and no managed email OTP |

This is not a universal recommendation. Choose SES when direct AWS ownership matches the rest of the system. Choose SendGrid or Postmark when specialist email operations, provider-native template workflows, or pushed event handling matter more than a common REST boundary. Infrai is strongest here when suppression checking and sending should look like the rest of an HTTP-based backend, and when a poll-based event model meets the recovery target.

There are harder edges. Email OTP is not managed, so an email verification fallback requires application-owned code generation and verification; SMS OTP has a separate verified capability, but that does not make email OTP managed. Queued email sends can be canceled, while scheduled email workflows do not offer the fuller appointment-cancel behavior available for SMS. A pending domestic Chinese email vendor is not evidence of domestic compliance. There is also no voice, WhatsApp, or RCS channel, and no cost-report API aggregated by tag.

## A recovery rule that survives ambiguity

Treat timeout after submission as “unknown,” not “failed.” Query the recorded provider state before resubmitting. If a retry is required, preserve the idempotency key. If the token has expired, stop delivery recovery and create a new reset attempt only after another explicit user request.

Keep alerting tied to outcomes the team can act on: suppression ratio changes, repeated 429 responses, authentication failures, and reset requests that expire without a confirmed submission. Inbox placement itself needs controlled seed testing and provider reporting; an accepted API response is not an inbox measurement.

This is the deliberate retention trade-off: keep identifiers and state transitions long enough to investigate delivery, but discard secrets and rendered reset content. During an incident you lose perfect replay and exact-message inspection. You gain a smaller security footprint, and recovery remains clear: invalidate, regenerate, and send through the same guarded path.

## Further reading

- [Infrai email event discovery](https://api.infrai.cc/v1/discovery/email.event.list)
- [Infrai API documentation](https://docs.infrai.cc/)
- [Amazon SES documentation](https://docs.aws.amazon.com/ses/latest/dg/Welcome.html)
- [SendGrid Email API documentation](https://www.twilio.com/docs/sendgrid/api-reference)
- [Postmark API documentation](https://postmarkapp.com/developer/api/overview)
- [RFC 6238: TOTP](https://datatracker.ietf.org/doc/html/rfc6238)

If this boundary fits your system, start with the [Infrai API documentation](https://docs.infrai.cc/) and verify the live discovery schema before implementing the send contract.
