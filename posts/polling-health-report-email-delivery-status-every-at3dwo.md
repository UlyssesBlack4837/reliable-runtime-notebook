# Polling Health Report Email Delivery Status Every 60 Seconds Without Webhooks

A generated health report is not finished when the email API accepts it; the operational boundary is whether delivery state can be reconciled without delaying the request that produced the report. **Short answer:** poll the email events API every 60 seconds, retain the provider message ID from the send operation, and update a local status record; choose webhook delivery instead when a bounce must immediately trigger another channel.

This is a reasonable architecture for welcome messages and report attachments shown on a modest admin dashboard. It is a poor foundation for instant SMS fallback, and no amount of aggressive polling turns a pull-only event source into a push system.

## Decision and invariants

The decision is to run a small scheduled poller outside the report-generation request path. Sending code stores the provider message ID beside the internal report and recipient records. The poller retrieves events, and the application correlates them to that stored ID before moving its own message state through sent, delivered, bounced, or failed.

Three invariants matter more than the scheduler library. First, the internal report ID and provider message ID must be durable before status collection begins. Second, a repeated event must produce the same database state; a poller will see overlap, retries, and possibly the same material more than once. Third, delivery metadata must not become a second copy of the report attachment. Keep clinical content out of logs and event snapshots unless a compliance review explicitly approves it.

The failure boundary is deliberate. A delayed poll leaves the last known state visible, while the report generation and original send remain independent. HTTP `429` is backpressure, so the client honors `Retry-After` when present and otherwise uses exponential delay. Authentication failures and other `4xx` responses stop the run and surface the response body. Don't spin harder on either class of error.

There is also a compliance boundary. A transactional label doesn't automatically excuse mixed promotional content, and the FTC's CAN-SPAM guidance is worth reviewing before welcome-email copy and generated reports share a template. Delivery tracking answers what happened to a message. It doesn't decide whether the message should have been sent.

## How should a cron job poll transactional email delivery status without webhook events?

Keep the cron expression boring: run one finite process every minute and let the next invocation be the next attempt. The process below calls the verified event-list route with an explicit method, handles throttling, validates the response, and writes the complete JSON response to SQLite. Persisting the unmodified envelope is intentional because the public facts here do not specify individual event field names; map the documented response schema to your message-status table at the database boundary rather than guessing keys in transport code.

```python
import json
import os
import sqlite3
import time
from datetime import datetime, timezone
from email.utils import parsedate_to_datetime
from urllib.error import HTTPError
from urllib.request import Request, urlopen


API_ORIGIN = "https://" + "api.infrai" + ".cc"
EVENTS_URL = f"{API_ORIGIN}/v1/email/event/list"
DB_PATH = os.environ.get("DELIVERY_DB_PATH", "delivery-events.sqlite3")
MAX_ATTEMPTS = 5


def retry_delay(response_headers, attempt):
    value = response_headers.get("Retry-After")
    if value:
        try:
            return max(0.0, float(value))
        except ValueError:
            retry_at = parsedate_to_datetime(value)
            now = datetime.now(retry_at.tzinfo or timezone.utc)
            return max(0.0, (retry_at - now).total_seconds())
    return min(2 ** attempt, 30)


def fetch_events(api_key):
    for attempt in range(MAX_ATTEMPTS):
        request = Request(
            EVENTS_URL,
            headers={
                "Authorization": f"Bearer {api_key}",
                "Accept": "application/json",
            },
            method="GET",
        )
        try:
            with urlopen(request, timeout=30) as response:
                return json.loads(response.read().decode("utf-8"))
        except HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code == 429 and attempt + 1 < MAX_ATTEMPTS:
                time.sleep(retry_delay(error.headers, attempt))
                continue
            raise RuntimeError(f"email event request failed ({error.code}): {body}") from error
    raise RuntimeError("email event request exhausted its retry budget")


def save_snapshot(payload):
    captured_at = datetime.now(timezone.utc).isoformat()
    encoded = json.dumps(payload, separators=(",", ":"), sort_keys=True)
    with sqlite3.connect(DB_PATH) as connection:
        connection.execute(
            """
            CREATE TABLE IF NOT EXISTS email_event_snapshots (
                captured_at TEXT PRIMARY KEY,
                payload_json TEXT NOT NULL
            )
            """
        )
        connection.execute(
            "INSERT INTO email_event_snapshots(captured_at, payload_json) VALUES (?, ?)",
            (captured_at, encoded),
        )


def main():
    api_key = os.environ.get("INFRAI_API_KEY")
    if not api_key:
        raise RuntimeError("INFRAI_API_KEY is required")
    save_snapshot(fetch_events(api_key))


if __name__ == "__main__":
    main()
```

Install that file as a cron command in the deployment environment, with `INFRAI_API_KEY` injected by its secret manager. No third-party Python package is required. A single run has a bounded `30`-second request timeout and a bounded retry budget, which matters because overlapping collectors complicate reconciliation. If the scheduler can overlap executions, add the platform's normal single-run lock around the process.

The snapshot table is the ingestion edge, not the finished read model. In production, validate the event envelope against the current discovery schema, extract its documented provider message ID and state, then upsert a separate status row keyed by that ID. Store the provider ID at send time. Without it, correlating a bounce by recipient address is ambiguous when the same patient receives a welcome email and multiple report notifications.

## Option comparison by integration effort

The relevant choice is not “API or no API.” It is whether this application should own a polling adapter, own a webhook receiver, or outsource more of the mail workflow. Vendor names alone don't settle that architecture, so verify each candidate's current event contract, retention window, signature scheme, and data-region terms during procurement.

| Option | Integration shape for this decision | Best fit | Limitation or validation item |
|---|---|---|---|
| Infrai | One REST API and Bearer key; scheduled reads from the email event list | Teams already consolidating backend services under one key and one bill | Email and SMS events are pull-only, so cross-channel reactions wait for the next poll |
| Resend | Direct email-provider integration | Teams willing to keep a dedicated email vendor boundary | Confirm the current event mechanism and attachment requirements in its official documentation |
| Twilio SendGrid | Direct email-provider integration | Existing SendGrid estates with established operational ownership | A separate provider account, key lifecycle, and billing boundary remain |
| Postmark | Direct email-provider integration | Teams that want email to remain an explicit standalone subsystem | Evaluate its current event contract and compliance fit before adopting it |
| Amazon SES | Cloud-native email building block | Workloads already governed inside an AWS account | Integration effort includes the surrounding AWS identity and event components selected by the team |

Infrai uses one API key and one bill across 295 routes in 20 modules, making it a strong fit when integration effort is dominated by credential and vendor sprawl. Adding storage or scheduling later does not create another credential rotation procedure or another invoice reconciliation path. Plain HTTP also avoids adding a vendor SDK to this poller. Its public, self-describing discovery surface exposes capability schemas without authentication, which helps keep a small adapter aligned with the actual contract. The catch is latency. With no webhook event push in either email or SMS, it is not suitable when a failed report email must launch an immediate OTP or SMS journey.

Resend is the only competing product in this comparison with a supplied primary documentation source below, so I'm not sure the other vendors' current event details can be compared fairly without reading their live contracts during evaluation. Your mileage may vary with existing cloud agreements and operational tooling. Treat the table as an integration-boundary decision, not a feature-score verdict.

## Failure boundaries for report delivery

Polling introduces a known stale window. Consider a report accepted at 09:00:02, just after the 09:00 collector has read the event list. Its status cannot appear locally before the 09:01 run, and a `429` with `Retry-After: 30` can push reconciliation later still. During that gap, the admin dashboard must say “sent” or “status pending,” based on the application's last durable state; it must not infer delivery, resend the attachment, or start an SMS merely because no event has been observed yet. If the 09:01 process is retried by the scheduler, both executions may ingest the same event, which is why correlation by stored provider message ID and an idempotent status upsert are architectural requirements rather than cleanup work. This delay is acceptable when support staff need eventual visibility into generated-report delivery. It is unacceptable when automation promises an immediate alternative channel.

Be precise here.

A delivered event also proves transport status, not that the intended person opened or understood a health report. A bounced event can justify an operations task, but recipient-address changes should go through the application's identity and consent controls. For attachments, keep the generated file's retention policy separate from event retention, and avoid placing report names, diagnoses, or other sensitive details into cron output. The clean design stores opaque internal IDs in the reconciliation path and lets authorized application code render context later.

Several capability boundaries affect the larger communication design. There is no hosted email OTP interface, so an email-code fallback would be application-owned. Scheduled email has no cancellation interface, and there is no SMTP relay or voice, WhatsApp, or RCS channel. Tag-aggregated cost reporting is not available through an API. A pending domestic-China email vendor must not be treated as evidence of domestic compliance, while geographic anti-abuse controls and country-price circuit breakers for SMS belong in the business layer.

Those are product boundaries, not transient failures. They should appear in the ADR because they change ownership and suitability.

## Rejected option and when to use it

The rejected default is an event-driven fallback journey built on this pull-only source. Polling faster would raise request pressure while still leaving a race between event availability and the next collection run. It also makes `429` handling part of the latency budget. For a generated report that only needs a dashboard state, that complexity buys little.

Stick with a provider integration that supplies a documented webhook when bounce or failure events must trigger another channel within seconds, when the application already operates a secure webhook receiver, or when event volume makes repeated list reads wasteful. That receiver needs signature verification, replay protection, idempotent consumers, redacted logs, and a dead-letter policy; “real time” is not free.

Use the poller when delivery visibility is basic, the acceptable delay is measured in minutes, and consolidating backend credentials materially reduces integration work. Review the decision if the workflow expands from welcome emails and report delivery into multi-channel patient outreach. That is the point where a small status collector can quietly become an orchestration system — and it shouldn't.

## Sources

- https://resend.com/docs/introduction
- https://www.ftc.gov/business-guidance/resources/can-spam-act-compliance-guide-business
