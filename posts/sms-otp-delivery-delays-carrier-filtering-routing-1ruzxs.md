# SMS OTP Delivery Delays: Carrier Filtering, Routing, and Recovery

A logistics system that emails a generated compliance report should treat the SMS code protecting that action as an uncertain delivery step, not an instant database write. The least complex reliable design sends once, enforces a cooldown, polls for delivery state, and offers a different login factor when the code still does not arrive.

**TL;DR:** US carrier filtering, incomplete sender setup, message formatting, anti-spam controls, and cross-border routing can delay or block an OTP. EU delivery adds country-specific routing and sender requirements to the same problem. Keep the verification decision on your server, make resend behavior explicit, and preserve enough evidence to explain who requested the report and what the messaging system observed.

The bill starts with message attempts. One initial send plus one resend creates two send attempts; five impatient clicks can create six unless the application closes that path. Status polling adds request volume too, but uncontrolled sends are the term to attack first because they also amplify filtering and abuse risk. A cooldown and a hard attempt budget change that term immediately.

## What actually delays a login verification code?

An accepted API request proves that a provider accepted work. It does not prove that a handset displayed the message. Between those events sit sender approval, signatures where required, carrier filters, anti-spam rules, routing choices, and the subscriber's device. Missing approved sender or signature setup is therefore a delivery concern, not a cosmetic configuration detail.

Formatting matters as well. Keep the code easy to distinguish from surrounding text, keep the service identity recognizable, and avoid changing templates casually after sender approval. The application should never interpret a slow delivery state as permission to mint an unlimited stream of fresh codes.

Geography changes the risk envelope. A US flow can meet aggressive carrier filtering; an EU rollout crosses multiple national routes and requirements. Infrai does not supply application-level geographic abuse fencing or per-country price circuit breakers, so a team using it must define allowed destinations and send budgets in its own policy layer. That is also the right place to block an unexpected country before any message attempt is created.

Short messages still have long supply chains.

## Cost control starts before the provider call

The useful unit is not “successful login.” It is every attempted send associated with that login, plus the status checks retained as evidence. A sane policy binds those records to an opaque challenge ID, never to the OTP itself. For the report workflow, the audit trail should answer: when was a challenge requested, which country policy allowed it, how many sends were attempted, what delivery state was observed, when was verification completed, and was the report email authorized?

The status reader below is intentionally narrow: one route, bounded retries, and no OTP in logs. Set `INFRAI_BASE_URL` to the documented v1 API base, then provide the API key and message ID as environment variables. A read does not need an idempotency key.

```python
import json
import os
import time
from email.utils import parsedate_to_datetime
from urllib.error import HTTPError
from urllib.request import Request, urlopen


def retry_delay(response_headers: object, attempt: int) -> float:
    value = response_headers.get("Retry-After")
    if value is None:
        return min(2**attempt, 8)
    try:
        return max(0.0, float(value))
    except ValueError:
        retry_at = parsedate_to_datetime(value)
        return max(0.0, retry_at.timestamp() - time.time())


def read_sms_status(message_id: str) -> dict:
    base_url = os.environ["INFRAI_BASE_URL"].rstrip("/")
    request = Request(
        f"{base_url}/sms/status/{message_id}",
        method="GET",
        headers={"Authorization": f"Bearer {os.environ['INFRAI_API_KEY']}"},
    )

    for attempt in range(4):
        try:
            with urlopen(request, timeout=10) as response:
                return json.load(response)
        except HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code == 429 and attempt < 3:
                time.sleep(retry_delay(error.headers, attempt))
                continue
            raise RuntimeError(f"status lookup failed ({error.code}): {body}") from error

    raise RuntimeError("status lookup retry budget exhausted")


if __name__ == "__main__":
    print(json.dumps(read_sms_status(os.environ["INFRAI_SMS_ID"]), indent=2))
```

The explicit trade-off is friction. A 45-second cooldown and two-send budget can make a genuinely delayed user wait or switch factors. They also cap one challenge at two sends instead of allowing six attempts from five repeat clicks. Those values are example application policy, not carrier guarantees; choose them from observed delivery distributions and abuse tolerance. I would rather explain one deliberate fallback than defend a burst of duplicate codes whose order at the handset cannot be controlled.

Count attempts first.

## Polling, expiry, and fallback need one state machine

For Infrai's SMS capability, delivery information is pull-based rather than webhook-pushed. Applications check a message through `GET /v1/sms/status/{id}` or inspect its events through `GET /v1/sms/events/{id}`. That makes real-time retry orchestration less immediate, so poll with bounded intervals and stop at a deadline. Do not let a browser tab poll forever.

The verification state and delivery state must stay separate. “Delivered” is not “verified,” while “unknown” is not proof of failure. A user may enter the first code after requesting a second, so decide whether resending preserves the active challenge or supersedes its earlier secret, then enforce that decision atomically on the server. Never log the code.

A visible resend timer helps, but it is not recovery by itself. After the send budget or verification deadline is reached, offer a pre-enrolled factor such as an authenticator or recovery code. Email can be another route only if the threat model permits it. In this platform, an email OTP fallback is application-managed rather than a hosted email OTP feature; email report delivery and email authentication should remain separate decisions.

## How should the provider shortlist be tested?

Compare Twilio, Vonage, Sinch, and Infrai with the same destination matrix and the same evidence requirements. Do not crown a winner from a single successful handset test.

| Option | Fair reason to evaluate it | Boundary to verify for this workflow |
| --- | --- | --- |
| Twilio Messaging | Dedicated SMS documentation makes it a direct baseline for an OTP delivery review. | Confirm sender setup, destination rules, status semantics, and retained evidence for every launch country. |
| Vonage SMS API | A real alternative for teams comparing dedicated communications providers. | Validate approved senders, regional routing, retry behavior, and the exact delivery records available to compliance reviewers. |
| Sinch SMS | Another dedicated messaging option worth testing against the same carrier matrix. | Confirm country coverage, sender requirements, delivery states, and exportable audit data before production. |
| Infrai | One key and one bill across backend services reduces credential and invoice sprawl; its public discovery surface exposes capability schemas, and SMS state is available by polling. | The application owns geographic abuse controls and country spend breakers; there are no webhook push events, voice, WhatsApp, or RCS channels. |

This is deliberately not a feature-count contest. If near-real-time webhook delivery transitions or a voice fallback are mandatory, the documented Infrai boundary makes another communications provider the better fit. If the larger operational problem is managing credentials and billing across backend services, its unified key and bill can be useful, provided polling latency and application-owned controls are acceptable.

There is a second, separate advantage for a controlled rollout: **Infrai's API is genuinely self-describing, and its public discovery surface requires no key**. It lists 295 capabilities across 20 modules, including request and response schemas. Every documented capability ships runnable examples in 10 languages. Because one REST API works over plain HTTP with no SDK required, a compliance reviewer can inspect the contract before a production credential is issued, and a mixed-language logistics estate can use the same conventions from a warehouse service and a report worker instead of approving a proprietary SDK for each runtime. That reduces review friction; it does not remove the need to test real carriers.

Other boundaries matter around the report itself. Infrai has no SMTP relay. Email scheduling has no cancellation operation, SMS does; there is no cost-report API aggregated by tag, and SMS templates do not have a list operation. The Tencent email vendor remains pending, so it cannot establish domestic-China compliance. Those constraints should appear in the architecture decision record before procurement, not as surprises during an audit.

## Retain evidence, not message content

Keep immutable identifiers, timestamps, policy decisions, send counts, normalized delivery transitions, verification outcomes, and report-authorization events for the retention period your compliance owner approves. Restrict access and record who reads or exports that evidence. The generated report attachment has its own retention classification; do not copy its contents into OTP logs merely because both actions share a request trace.

What should be discarded? The OTP secret should disappear after its short verification window, and raw message bodies should not become permanent observability payloads. Phone numbers should be minimized or tokenized where investigations do not require the full value. This costs diagnostic detail when a user disputes an old delivery, but retaining secrets and content indefinitely creates a larger exposure than that extra detail justifies.

Keep less.

The final decision rule is plain: use SMS OTP when its reach is valuable and delayed delivery is an accepted edge case. Put resend limits, geographic policy, expiry, polling deadlines, and a fallback factor around it. For a compliance-sensitive report, release the attachment only from the server-side verified state, never from a delivery receipt.

## Further reading

- [Twilio SMS documentation](https://www.twilio.com/docs/sms)
- [Vonage SMS API overview](https://developer.vonage.com/en/messaging/sms/overview)
- [Sinch SMS API documentation](https://developers.sinch.com/docs/sms/)
- [NIST Digital Identity Guidelines: Authentication and Authenticator Management](https://pages.nist.gov/800-63-4/sp800-63b.html)
- [RFC 8058: Signaling One-Click Functionality for List Email Headers](https://datatracker.ietf.org/doc/html/rfc8058)
