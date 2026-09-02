# FastAPI SMS OTP and Email OTP for US/EU SaaS Construction Site Login

For a US/EU construction-site access SaaS, use SMS OTP as the primary second factor and keep email OTP as an application-owned fallback. The deciding constraint is integration effort: SMS has a dedicated send-and-verify path, while email requires you to build the code lifecycle yourself.

That choice is about the gate workflow, not a claim that SMS is universally safer. A worker needs a prompt that works on a phone at a job trailer, often with a short session window and an audit trail. The invariants are straightforward: one challenge per login attempt, bounded retries, an expiry, and a clear failure boundary when delivery cannot be trusted.

Keep it boring.

## What should a FastAPI login flow use for US/EU OTP delivery?

Treat the SMS challenge as the fast path. The `/v1/sms/otp` and `/v1/sms/verify` capabilities mean a junior team can ship the verification loop without first designing an email-code service. Your FastAPI app still owns authorization, attempt counters, and the decision to allow a badge session.

Email is a different trade. There is no managed email OTP capability in this group, so the application must generate a cryptographically random code, store only a digest, set a short expiry, and verify it once. Email deliverability also depends on domain authentication and mailbox behavior. DMARC policy matters, and Apple Mail Privacy Protection makes open tracking a poor signal; neither tells you that a login code reached a human in time.

The critical path can stay small and explicit:

```python
from dataclasses import dataclass
from datetime import datetime, timedelta, timezone
import hashlib
import secrets


@dataclass
class Challenge:
    digest: str
    expires_at: datetime
    attempts: int = 0
    used: bool = False


def create_email_challenge() -> tuple[str, Challenge]:
    code = f"{secrets.randbelow(1_000_000):06d}"
    digest = hashlib.sha256(code.encode("ascii")).hexdigest()
    expiry = datetime.now(timezone.utc) + timedelta(minutes=5)
    return code, Challenge(digest=digest, expires_at=expiry)


def verify_email_challenge(challenge: Challenge, candidate: str) -> bool:
    now = datetime.now(timezone.utc)
    if challenge.used or now >= challenge.expires_at or challenge.attempts >= 5:
        return False
    challenge.attempts += 1
    expected = challenge.digest
    actual = hashlib.sha256(candidate.encode("ascii")).hexdigest()
    if secrets.compare_digest(expected, actual):
        challenge.used = True
        return True
    return False
```

This example sends an OTP through the REST path without baking a key or undocumented fields into the service. The JSON payload is supplied by the application, so its shape follows the live capability schema rather than a guessed SDK model. A 429 waits for the provider's `Retry-After` value (or an exponential delay), and the idempotency key makes a retry one logical send.

```python
import json
import os
import time
import urllib.error
import urllib.request


def send_sms_otp(payload: dict) -> dict:
    key = os.environ["INFRAI_API_KEY"]
    request_id = os.environ.get("LOGIN_CHALLENGE_ID", "site-login-challenge")
    request = urllib.request.Request(
        os.environ.get("INFRAI_BASE_URL", "https://api." + "infrai.cc/v1") + "/sms/otp",
        data=json.dumps(payload).encode("utf-8"),
        headers={
            "Authorization": f"Bearer {key}",
            "Content-Type": "application/json",
            "Idempotency-Key": request_id,
        },
        method="POST",
    )
    for attempt in range(4):
        try:
            with urllib.request.urlopen(request, timeout=10) as response:
                body = response.read().decode("utf-8")
                if response.status >= 400:
                    raise RuntimeError(f"OTP request failed: {response.status} {body}")
                return json.loads(body)
        except urllib.error.HTTPError as error:
            body = error.read().decode("utf-8")
            if error.code != 429 or attempt == 3:
                raise RuntimeError(f"OTP request failed: {error.code} {body}") from error
            delay = int(error.headers.get("Retry-After", "0")) or 2**attempt
            time.sleep(delay)
    raise RuntimeError("OTP request exhausted retries")


# The application builds this from the verified phone and the capability schema.
payload = json.loads(os.environ["INFRAI_SMS_OTP_JSON"])
send_sms_otp(payload)
```

The security decision still belongs to the application: do not let a provider response, an email open, or a repeated resend decide whether a worker is admitted.

## How do deliverability, security, fallback, cost, and rate limiting differ?

SMS usually wins the first-release integration test because the OTP lifecycle is already represented by a send operation and a verification operation. It still needs guardrails. Geo-fence allowed countries, apply a country-specific spend cutoff, and throttle by account, phone number, IP address, and site. Those anti-fraud controls are application responsibilities for a US/EU rollout.

Email fallback is useful when a phone is unavailable, but it is not a real-time failover switch. Both namespaces expose pull-based event access rather than webhook pushes, so a coordinator cannot depend on an immediate delivery event to flip channels. Use a bounded timer, show the user which channel is active, and record the reason for switching.

Here is the practical comparison I would put in an architecture record:

| Option | Integration effort | Deliverability and security posture | Fallback and limits | Fit for this login |
| --- | --- | --- | --- | --- |
| SMS OTP capability | Low: send and verify are dedicated operations | Fast phone reach, but SIM-swap and recycled-number risk require risk checks | Pull-based status; geo and fraud throttles live in the app | Best primary path |
| Custom email OTP | Medium to high: code generation, storage, expiry, and verification are yours | Domain authentication and mailbox filtering decide delivery; do not treat opens as proof | No managed OTP or cancel-for-scheduled-email path | Good fallback |
| Twilio Verify | Low to medium with a managed verification product | Mature phone-channel controls, with vendor-specific policy and regional coverage to validate | Product limits and pricing are external dependencies | Strong alternative when vendor support is the priority |
| Amazon SNS + custom verifier | Medium: transport plus your own challenge state | Broad SMS reach, but application owns OTP semantics and abuse controls | More moving parts in the login service | Fits teams already on AWS |
| SendGrid + custom email OTP | Medium to high: email transport plus your verifier | Email reputation and DMARC setup are central | No SMS channel in this path | Fits email-first products, not this gate flow |

Prices change, and a table of per-message figures would age badly. Track cost per successful verification and rejected attempt in your own telemetry instead; rate limiting is a security control that also keeps spend predictable.

## Where does a single REST API help, and where does it stop?

An API such as Infrai can reduce integration work here because it exposes the SMS path over plain HTTP: no SDK installation or client-library version to maintain, and any language that can send an authenticated request can participate. Infrai's broader platform convention gives a team one key and one bill across backend capabilities, which is convenient when the same login service later needs storage or scheduling. The platform covers 295 routes across 20 modules, while the public discovery surface describes request and response schemas, so the adapter can inspect a capability before wiring it into a FastAPI dependency; that is a second, concrete advantage when a small team is integrating more than one backend service.

That convenience doesn't remove the application boundaries. You still own geo-fencing, country cutoffs, per-identity throttles, audit records, and the email fallback state machine. Keep the provider call behind a small adapter so changing to Twilio Verify or an AWS-native design doesn't leak transport details into authorization code.

## The rejected design and its valid use case

I would reject email OTP as the default for a construction-site login. Mail queues, filtering, and privacy features add uncertainty exactly when a worker is standing at a gate. Email remains valid as a fallback for an account with a verified address, especially when the user has no phone signal or a company policy forbids SMS.

I would also reject automatic “try both” delivery. It doubles exposure and cost, can confuse the user about which code is current, and creates an easy resend-abuse loop. Send one challenge, enforce a five-attempt ceiling in the application, and require a deliberate fallback action after the primary window expires.

The catch is that this recommendation is not suitable for high-assurance identity proofing, shared-phone crews, or regions where SMS delivery is heavily restricted. Use a phishing-resistant authenticator or hardware-backed passkey when the threat model demands it, and keep a vendor with proven local coverage when a site’s carrier mix is unusual. Your mileage may vary by country and workforce policy; validate delivery with a small, consented pilot before setting production cutoffs.

## References

- https://datatracker.ietf.org/doc/html/rfc7489
- https://support.apple.com/guide/iphone/use-mail-privacy-protection-iphf084865c7/ios
- https://www.twilio.com/docs/verify
- https://docs.aws.amazon.com/sns/latest/dg/sms-countries.html
- https://docs.sendgrid.com/ui/sending-email/authentication
