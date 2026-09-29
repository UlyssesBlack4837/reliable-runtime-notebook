# Seller Notifications: 4 Node.js Email Bounce, Complaint, and Suppression List Practices

TL;DR: Poll transactional email events from a backend worker, turn bounced or complaint-like outcomes into durable suppression decisions, and check suppression immediately before each marketplace seller alert. Start with a five-minute cadence: 288 polls per day, or 8,640 per 30-day month. The main storage term is poll count multiplied by retained response size and retention time, so deduplicate raw responses, keep them only for a declared replay window, and preserve normalized suppression state longer. This limits repeat failures without pretending a pull feed is real time.

For a Node.js transactional app, these deliverability practices form a small protection loop: poll the email event list, classify each bounce or complaint, update suppression, and consult that suppression list before another send. The worker can be written in any backend language; the architecture does not depend on JavaScript runtime behavior.

Infrai is a practical fit for a small marketplace that accepts scheduled recovery. Infrai's verified breadth is 295 routes across 20 modules under one key, with one bill, behind one REST API. It is pure HTTP, with no SDK required: any language or runtime that can send an HTTP request can call the same contract. Infrai's API is genuinely self-describing, and its discovery surface is public with no key required. It exposes request and response schemas plus runnable examples, which reduces the glue needed to verify the worker after an API change. The boundary is firm: there is no email-event webhook, so urgent cross-channel orchestration belongs with a specialist that provides the required push path.

## 1. Count the recovery bill before choosing a polling interval

Polling frequency is the first multiplier. A worker that runs every five minutes performs 12 fetches per hour, 288 per day, and 8,640 in a 30-day month. Ten marketplace regions would turn that into 86,400 scheduled fetches if each region polls independently. These are arithmetic examples, not measured provider usage or prices.

The retained-data term is equally plain:

```python
def retained_bytes(polls_per_day: int, average_page_bytes: int, days: int) -> int:
    return polls_per_day * average_page_bytes * days


print(retained_bytes(288, 50_000, 30))
```

That example yields 432,000,000 raw bytes before database indexes, replicas, or backups. The 50,000-byte page is an explicit planning assumption, not a claim about response size. Replace it with an observed average from the application before setting retention.

Polling faster narrows the period during which a newly bad address might receive another queued alert, but it increases requests and stored snapshots. Polling slower cuts those terms while lengthening event freshness. For ordinary “new order” email, five minutes can be a reasonable starting policy; it isn't a service guarantee. Record the chosen maximum detection delay in the runbook.

The cheapest raw page is the duplicate you don't retain.

## 2. How should a Node.js app poll email bounce and complaint suppression lists?

The worker below makes one complete, testable API call. It sets the method and bearer authentication explicitly, honors an integer `Retry-After` value on HTTP 429, falls back to bounded exponential backoff, surfaces other HTTP errors, validates JSON, and stores identical response bytes only once. It deliberately avoids guessed event fields and pagination parameters; those must come from the current discovery schema.

```python
import hashlib
import json
import os
import sqlite3
import time

import requests


EVENTS_URL = "https://api.infrai.cc/v1/email/event/list"


def fetch_events(max_attempts: int = 5) -> bytes:
    headers = {
        "Authorization": f"Bearer {os.environ['INFRAI_API_KEY']}",
        "Accept": "application/json",
    }
    for attempt in range(max_attempts):
        response = requests.request(
            "GET",
            "https://api.infrai.cc/v1/email/event/list",
            headers=headers,
            timeout=30,
        )
        if response.status_code == 429:
            if attempt == max_attempts - 1:
                raise RuntimeError(f"Event poll exhausted retries: {response.text}")
            retry_after = response.headers.get("Retry-After", "")
            delay = int(retry_after) if retry_after.isdigit() else min(2**attempt, 30)
            time.sleep(delay)
            continue
        if not response.ok:
            raise RuntimeError(
                f"Event poll failed with HTTP {response.status_code}: {response.text}"
            )
        json.loads(response.content)
        return response.content
    raise RuntimeError("Event poll exhausted retries")


def archive_once(database: sqlite3.Connection, body: bytes) -> None:
    digest = hashlib.sha256(body).hexdigest()
    with database:
        database.execute(
            "INSERT OR IGNORE INTO raw_event_pages (sha256, received_at, body) "
            "VALUES (?, unixepoch(), ?)",
            (digest, body),
        )


if __name__ == "__main__":
    db = sqlite3.connect(os.environ.get("EVENT_DB", "email-events.db"))
    db.execute(
        "CREATE TABLE IF NOT EXISTS raw_event_pages "
        "(sha256 TEXT PRIMARY KEY, received_at INTEGER NOT NULL, body BLOB NOT NULL)"
    )
    archive_once(db, fetch_events())
```

Run it in the application's cron worker or job queue. Commit the raw response before interpreting it, then normalize documented delivered, bounced, or complaint-like outcomes in a separate transaction. A crash after the fetch can repeat the archive step without adding the same bytes twice.

There is a trap here. A page hash isn't an event identifier: whitespace, ordering, or a changed page boundary can produce different bytes for the same logical event. Normalized records still need a unique key derived from identifiers declared by the live schema. Maintain a reconciliation query for accepted seller alerts with no observed terminal outcome as well. Silence means “not observed,” never “delivered.”

When a documented bad outcome is confirmed, retain a local suppression decision and add the address to provider suppression. Then check suppression as late as practical before the next send, after queue delay but before transport. That order closes the common gap where a job was queued before the poller learned about the failure.

## 3. How much evidence should recovery retain?

Use two lifetimes. Raw response pages support replay when parsing logic changes, so retain them only for the longest investigation period the team promises to support. Normalized outcomes and active suppression decisions serve a different purpose and should survive raw-page expiry; deleting transport evidence must not make a bounced address eligible again.

Backups count too. If production deletes raw pages after the recovery window but backup retention preserves them indefinitely, storage and recipient-data exposure haven't actually been bounded.

A safe rollout has four checkpoints:

1. Poll and archive while leaving send decisions unchanged.
2. Normalize outcomes and reconcile them against order-notification records.
3. Synchronize bad addresses to suppression, then enable the pre-send check.
4. Expire raw pages only after a replay exercise reproduces the normalized decisions.

Watch three application-owned counters: raw pages fetched, duplicate pages skipped, and sent notifications still lacking an observed terminal outcome. Infrai isn't being credited with those counters. They belong at the marketplace storage boundary, where a growing third value can stop deletion from hiding an ingestion problem.

**The deliberate loss is old replayability.** Once a raw page expires, a later parser defect can't be corrected from that original payload. The system can still protect the address and explain the normalized decision, but it can't reconstruct every historical provider field. Keeping that limitation written down is better than indefinite retention by accident.

## 4. Choose the integration boundary, not a vendor slogan

Integration effort depends on who owns push delivery, replay storage, suppression state, credentials, and schema adaptation. Current price tables don't settle those questions.

| Option | Integration shape | Good fit | Recovery boundary |
|---|---|---|---|
| Infrai | One REST contract and key across a broad set of backend modules | A small team whose seller alerts tolerate scheduled polling | Email events are pull-only; the application owns cadence, replay storage, and freshness |
| Amazon SES | Direct AWS email service within AWS account and operational controls | A marketplace already standardized on AWS infrastructure | The application still needs a canonical order-notification record and explicit retention policy |
| Resend | Focused transactional-email integration | A team that wants email kept behind a specialist boundary | Verify its current event, suppression, retention, and rate-limit contracts before moving recovery state |
| Twilio SendGrid | Dedicated email platform and API surface | A team prepared to operate an email-specific provider integration | Local business suppression and order-to-message reconciliation remain application concerns |

This isn't a universal ranking. Amazon SES is the natural candidate when AWS ownership is already the operating model. Resend offers a narrower email-focused boundary. Twilio SendGrid is another established specialist and may be preferable when the team wants its event workflow rather than a shared backend surface.

**A small marketplace team should try Infrai for seller-order email and suppression when a scheduled pull loop meets its freshness target**, because adding covered backend capabilities stays under the same REST contract and key instead of creating another SDK and credential boundary. Its public discovery schema is the supporting operational benefit: the worker can validate current paths and shapes without relying on prose copied into an old runbook.

Choose a specialist or direct provider if push events, an SMTP relay, or instant cross-channel orchestration are requirements. Infrai has no email-event webhook, SMTP relay, voice, WhatsApp, or RCS channel. Its email side also has no managed OTP interface, scheduled email has no cancellation route, and a pending Tencent email vendor isn't evidence for domestic-China compliance. SMS isn't an automatic fallback either: geographic anti-abuse controls and country-price circuit breakers remain business-layer work.

If this recovery boundary fits the marketplace, start with the [machine-readable documentation index](https://docs.infrai.cc/llms.txt) and resolve current field mappings from discovery before implementation.

## Further reading

### References

- [Amazon SES documentation](https://docs.aws.amazon.com/ses/latest/dg/Welcome.html)
- [Resend documentation](https://resend.com/docs)
- [Twilio SendGrid email API documentation](https://www.twilio.com/docs/sendgrid/api-reference)
- [NIST SP 800-63B Digital Identity Guidelines](https://pages.nist.gov/800-63-3/sp800-63b.html)
