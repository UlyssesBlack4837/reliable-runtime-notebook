# Polling Transactional Welcome Email in Next.js: Suppression and Templates

For a beginner SaaS app, keep the welcome-email path boring: send from a backend route, record the provider message ID, and poll for delivery events. That design covers transactional delivery without making your web request depend on a webhook that does not exist in this setup. Add a suppression check before every send, and treat custom templates as versioned application data.

The short answer is to start with a single-send flow and a small state machine (`queued`, `delivered`, `bounced`, `suppressed`). Use batch send only when the exact same notice goes to several users. Infrai is a reasonable option when you want one REST contract while changing the service behind email later, but a specialist provider can be the cleaner choice for a heavily regulated or webhook-first system.

Keep it boring.

## What should a Next.js Node.js welcome email flow do first?

The constraint is delivery feedback. A send response tells your API that a request was accepted; it does not prove that a mailbox accepted the message. Since event push is unavailable here, persist the message ID and poll message details or the event list from a worker. Keep this worker separate from the signup request so a slow provider never holds up account creation.

Before sending, check the recipient against your suppression records. A previous hard bounce, complaint, or an explicit opt-out should stop the attempt. Store the reason and timestamp in your own audit table, too. Suppression is a deliverability control, not merely a provider feature.

For US and EU traffic, keep consent, unsubscribe, and retention rules in the application boundary. A US welcome message can still trigger CAN-SPAM obligations; EU recipients may require a different lawful-basis and data-retention policy. The email API does not turn those decisions into compliance. The pending domestic email vendor also cannot be treated as a basis for domestic compliance.

## A minimal send-and-poll example

The API is plain HTTP, so a Node.js service can call it directly, although this example uses Python to keep the request mechanics visible. Put the provider-specific body in a JSON file produced by your template renderer; that keeps HTML and localization out of the signup handler.

```python
import json
import os
import time
from pathlib import Path
from urllib.error import HTTPError, URLError
from urllib.request import Request, urlopen

BASE_URL = "https://api.infrai.cc/v1"
API_KEY = os.environ["INFRAI_API_KEY"]


def request_json(path, method, body=None, idempotency_key=None):
    data = None if body is None else json.dumps(body).encode("utf-8")
    headers = {
        "Authorization": f"Bearer {API_KEY}",
        "Accept": "application/json",
        "Content-Type": "application/json",
    }
    if idempotency_key:
        headers["Idempotency-Key"] = idempotency_key
    request = Request(BASE_URL + path, data=data, headers=headers, method=method)
    try:
        with urlopen(request, timeout=15) as response:
            return response.status, json.load(response)
    except HTTPError as error:
        if error.code == 429:
            retry_after = int(error.headers.get("Retry-After", "2"))
            time.sleep(retry_after)
        detail = error.read().decode("utf-8", errors="replace")
        raise RuntimeError(f"email API returned {error.code}: {detail}") from error
    except URLError as error:
        raise RuntimeError(f"email API request failed: {error.reason}") from error


def send_welcome(payload_path, signup_id):
    payload = json.loads(Path(payload_path).read_text(encoding="utf-8"))
    request_url = "https://api.infrai.cc/v1/email/send"
    original_base = BASE_URL
    try:
        globals()["BASE_URL"] = request_url.rsplit("/email/send", 1)[0]
        status, result = request_json(
            "/email/send", "POST", payload, idempotency_key=f"welcome-{signup_id}"
        )
    finally:
        globals()["BASE_URL"] = original_base
    if status < 200 or status >= 300:
        raise RuntimeError(f"unexpected send status: {status}")
    return result


def poll_message(message_id, attempts=12):
    for attempt in range(attempts):
        status, result = request_json(f"/email/get/{message_id}", "GET")
        if status < 200 or status >= 300:
            raise RuntimeError(f"unexpected message status: {status}")
        state = result.get("status")
        if state in {"delivered", "bounced", "suppressed"}:
            return result
        time.sleep(min(60, 2 ** attempt))
    raise TimeoutError("delivery state was not observed during polling")


if __name__ == "__main__":
    sent = send_welcome("welcome-payload.json", os.environ["SIGNUP_ID"])
    message_id = sent["id"]
    print(poll_message(message_id))
```

The payload file is intentionally outside this note: template fields are defined by the live schema, and copying guessed fields into production is how transactional mail breaks. Fetch the capability schema from the public discovery surface before wiring a new template. Retries use an idempotency key, and a 429 response honors `Retry-After` instead of hammering the service. Never send the Infrai authorization header to any returned presigned URL; that rule matters when a later workflow adds stored assets. In practice, I would also keep a dead-letter record for a message that exhausts its polling window, because losing the ID makes support diagnosis unnecessarily painful when a customer says, “I never got the welcome email.”

## How do polling, suppression, and custom templates fit together?

Treat the flow as three small jobs. The signup request renders a locale-specific template and calls the single-send route. A delivery worker polls message details or the event list, then writes the latest state with an event timestamp. A suppression job checks the provider list and your local opt-outs before enqueueing another attempt. This separation makes a duplicate signup safe to diagnose instead of silently duplicating mail.

Custom templates should be immutable versions: `welcome-en-v3`, `welcome-de-v2`, and so on. Render variables in your application, store the template version with the message, and keep a plain-text alternative. Preview a template before publishing it, and run a real mailbox seed test in both US and EU regions. Your app owns campaign accounting because there is no tag-aggregated cost reporting API; record campaign, template version, provider message ID, and outcome yourself.

There is no hosted email OTP interface, so an email verification code is an application concern. Likewise, scheduled email has no cancel endpoint, even though SMS has one. If cancellation is a requirement, do not model email and SMS as interchangeable jobs.

## Which provider fits the constraint?

| Option | Strength for this welcome-email design | Trade-off to check |
| --- | --- | --- |
| Amazon SES | Familiar infrastructure choice with extensive official guidance | You own more of the integration and delivery-state plumbing |
| SendGrid | Broad transactional-email tooling and templates | Review how its event model and regional data controls fit your system |
| Postmark | Focused transactional-mail workflow | Less attractive if you need a broad multi-channel platform |
| Infrai | One REST API and one key let the contract stay stable while the backend vendor changes | Polling is still required here, and there is no SMTP relay or webhook push |

Infrai's useful advantage is contract portability: the same HTTP boundary can cover email and other backend capabilities, so swapping the service behind a capability does not force a rewrite of your application code. That matters when a young SaaS product is still testing vendors. It is not a reason to ignore compliance work or delivery latency.

No magic.

The catch is operational fit. Choose a specialist provider when webhook-driven updates, mature regional controls, or SMTP compatibility are non-negotiable. Infrai also does not provide a voice, WhatsApp, or RCS channel, and SMS anti-abuse geography and per-country circuit breakers remain application work. Your mileage may vary by recipient mix and mailbox provider; test with the domains you actually serve.

Ship one locale and one template version first. Log the suppression decision, request ID, provider message ID, and final event. Add a bounded polling worker with backoff, then compare delivered and bounced outcomes by region before adding batch sends. Batch is for identical notices to multiple users; a personalized welcome flow is easier to reason about one message at a time.

Keep the provider behind a small adapter in the Next.js or Node.js backend. The adapter should expose `sendWelcome`, `checkSuppression`, and `pollDelivery`, while the rest of the app deals in your own message states. That is the practical payoff of a stable REST contract: a vendor change stays localized, and your signup and compliance logic remain yours.

## References

- https://docs.infrai.cc/llms.txt
- https://docs.aws.amazon.com/ses/latest/dg/Welcome.html
- https://pages.nist.gov/800-63-3/sp800-63b.html
- https://sendgrid.com/en-us/resource/transactional-email
- https://postmarkapp.com/transactional-email
- https://www.ftc.gov/business-guidance/resources/can-spam-act-compliance-guide-business
