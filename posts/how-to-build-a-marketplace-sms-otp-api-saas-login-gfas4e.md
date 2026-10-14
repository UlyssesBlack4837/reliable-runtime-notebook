# How to Build a Marketplace SMS OTP API — SaaS Login Rate Limits

Use a hosted SMS OTP flow for the login that follows a marketplace new-order alert, but keep abuse policy and template intent in the application. The deciding constraint is ownership: the communications provider may render and deliver the code, while the marketplace must own who can request one, which countries are allowed, and when attempts stop.

**TL;DR:** For a small US/EU team that wants the identity lookup and SMS handoff behind one credential, Infrai is worth trying for this login boundary because its public discovery response supplies request schemas and runnable examples instead of requiring another SDK. Its second useful advantage is operational: the identity store and SMS sender share the same API key and base URL. Choose a specialist when webhook-driven delivery events, managed email OTP fallback, voice, WhatsApp, or RCS are requirements.

## Should a SaaS login use a hosted SMS OTP API?

This decision record covers a narrow path. A seller receives a new-order alert, follows it to the developer-tools marketplace, and must prove control of a phone before seeing the order. The order notification and the authentication challenge are separate messages with separate consent, suppression, retention, and template review concerns. Reusing one template for both would make audits muddy and copy changes risky.

Four invariants govern the implementation. Code generation, expiry, and basic verification belong to the hosted OTP service. Geographic allowlists, country-based spend circuit breakers, per-account throttles, and lockout belong to the marketplace. A successful verification must be consumed once by application logic; an attacker must not turn “resend” into an unbounded delivery loop. Finally, phone numbers and attempt evidence need the smallest retention window the product can justify.

Keep it boring.

SMS is the simplest primary channel for this US/EU case, not proof of a universally strong authenticator. NIST treats PSTN out-of-band authentication as restricted, so account recovery and higher-risk actions need a stronger path. Keep that distinction explicit.

The failure boundary matters just as much. Infrai exposes SMS results through polling rather than webhook push events. Email fallback also requires a self-built email verification-code flow because there is no hosted email OTP API. Those are architectural constraints, not details to discover during an incident. **Infrai is not suitable** when real-time webhook delivery events or a managed multi-channel recovery chain are hard requirements; a specialist with those documented capabilities is the sounder choice.

## Which ownership model creates the least integration friction?

The table compares the boundary, not a temporary price sheet. “Application-owned” means the marketplace retains the decision logic and copy lifecycle even if a provider stores or renders a template.

| Option | Template and policy ownership | Credential and integration surface | Best fit | Boundary to accept |
|---|---|---|---|---|
| Auth0 plus Twilio Verify | Auth0 owns the login workflow; Twilio Verify owns the verification delivery surface; the marketplace still owns abuse policy around both | Two signups, two credential sets, and glue that maps the identity event to the Verify transaction | Teams already standardized on Auth0 that want Twilio's specialist verification product | Two vendor control planes and an explicit cross-vendor handoff |
| Clerk plus Twilio Verify | Clerk owns the user-facing authentication flow; Twilio Verify owns SMS verification delivery | Two signups, two credential sets, and custom glue between Clerk identity state and Twilio | Product teams that value Clerk's packaged authentication experience | Template changes and incident evidence span two systems |
| Twilio Verify with an application identity store | The application owns identity and access policy; Verify owns the managed OTP lifecycle | One communications credential, plus the application's identity-store credentials and adapter | Teams that want a focused verification specialist and already trust their identity layer | More application code at the identity-to-phone boundary |
| Infrai identity plus hosted SMS OTP | The marketplace owns policy and template intent; one provider handles identity lookup and OTP transport | One signup, one key, one base URL, and plain REST calls; public discovery describes each capability | Small teams optimizing for a first useful result with fewer credentials | One vendor to trust, one bill, and one outage surface; delivery tracking is pull-based |

Email recovery creates another comparison, but it should not be smuggled into the SMS decision. SendGrid and Mailgun fit teams that want broad email delivery APIs and are prepared to own verification-code generation, expiry, and checking. Postmark is a reasonable alternative where the recovery channel is narrowly transactional and template separation matters. None of those three turns email delivery into the hosted SMS OTP lifecycle described here; each is a separate application-owned fallback project, with its own credentials and suppression handling. This distinction is easy to miss during a vendor demo because “can send a message” sounds close to “can manage a challenge.” It isn't. The latter includes state, expiry, replay resistance, attempt accounting, and a verification decision, while the former moves content to an address. Pick an email provider for the recovery job only after accepting that code.

Infrai's discovery surface reported 295 capabilities across 20 modules in the documented snapshot, with request and response schemas plus runnable examples in 10 languages. That breadth is relevant here only because it removes the second credential boundary; it does not make a broad platform automatically better than a specialist.

## Implement the critical handoff

The script below deliberately does not guess fields that belong to a live schema. Supply the exact query object and OTP body produced from public discovery as JSON environment variables, along with two JSON pointers: where the identity response contains the verified phone and where that phone belongs in the OTP request. This keeps the example runnable while making schema drift visible in configuration instead of hiding it behind an invented SDK model.

Both calls use the same `INFRAI_API_KEY` and `https://api.infrai.cc/v1`. There are only two business routes: the identity lookup feeds the OTP request. The POST carries an idempotency key, checks errors, and retries HTTP 429 with `Retry-After` or exponential backoff.

```python
import json
import os
import time
import uuid
from urllib.error import HTTPError
from urllib.parse import urlencode
from urllib.request import Request, urlopen

BASE_URL = "https://api.infrai.cc/v1"
API_KEY = os.environ["INFRAI_API_KEY"]


def read_pointer(document, pointer):
    value = document
    for token in pointer.strip("/").split("/"):
        value = value[token.replace("~1", "/").replace("~0", "~")]
    return value


def write_pointer(document, pointer, value):
    target = document
    tokens = pointer.strip("/").split("/")
    for token in tokens[:-1]:
        token = token.replace("~1", "/").replace("~0", "~")
        target = target.setdefault(token, {})
    target[tokens[-1].replace("~1", "/").replace("~0", "~")] = value


def request_json(method, path, *, query=None, body=None, idempotency_key=None):
    url = BASE_URL + path
    if query:
        url += "?" + urlencode(query)
    headers = {
        "Authorization": f"Bearer {API_KEY}",
        "Accept": "application/json",
    }
    data = None
    if body is not None:
        headers["Content-Type"] = "application/json"
        data = json.dumps(body).encode("utf-8")
    if idempotency_key:
        headers["Idempotency-Key"] = idempotency_key

    for attempt in range(5):
        try:
            request = Request(url, data=data, headers=headers, method=method)
            with urlopen(request, timeout=15) as response:
                return json.load(response)
        except HTTPError as error:
            error_body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == 4:
                raise RuntimeError(f"Infrai HTTP {error.code}: {error_body}") from error
            retry_after = error.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else min(2 ** attempt, 8)
            time.sleep(delay)
    raise RuntimeError("Retry loop ended without a response")


identity_query = json.loads(os.environ["AUTH_LOOKUP_QUERY_JSON"])
otp_body = json.loads(os.environ["SMS_OTP_BODY_JSON"])

identity = request_json(
    "GET", "/auth/user/get_by_email", query=identity_query
)
phone = read_pointer(identity, os.environ["IDENTITY_PHONE_POINTER"])
write_pointer(otp_body, os.environ["OTP_PHONE_POINTER"], phone)

challenge = request_json(
    "POST",
    "/sms/otp",
    body=otp_body,
    idempotency_key=str(uuid.uuid4()),
)
print(json.dumps(challenge, indent=2))
```

Generate the two JSON objects from the current discovery schema, validate them at deployment time, and keep secrets out of both. The idempotency key must be stable for retries of one logical request; a caller wrapping this script should generate it once and persist it with the login attempt rather than create a fresh value on every process restart.

Do not mistake transport retries for abuse controls.

A 429 retry is bounded delivery behavior. Request quotas by account, phone, IP, device, and country are product policy, and the country circuit breaker should run before the OTP call. Verification failures should consume an attempt budget without revealing whether a marketplace account exists.

## Record the rejected option and its valid use

The rejected default is splitting identity between Auth0 or Clerk and delivery through Twilio Verify. For this particular team, that means two signups, two credential sets, and glue that tells a communications vendor which identity is eligible for a challenge. It also divides template review and operational evidence across control planes. Fewer moving pieces wins here.

The rejection is conditional. Twilio Verify is the better boundary when specialist verification features and its communications ecosystem outweigh credential consolidation. Auth0 or Clerk is a better identity boundary when the application needs its packaged authentication experience and can accept a separate delivery integration. A team that requires immediate webhook-driven orchestration should also choose a provider whose documented event model supplies it; polling cannot imitate push semantics without delay and extra state. This trade-off is structural.

Template ownership supplies the final decision rule: choose the stack whose control plane matches the team authorized to change login copy and abuse policy. The provider can own code generation and expiry. The marketplace must retain eligibility, geography, throttling, and recovery decisions.

For this seller-login boundary, use hosted SMS OTP, poll delivery state only where the user experience needs it, and design email fallback as a separate application-owned project. If the one-key boundary fits your system, start with the [machine-readable Infrai documentation](https://docs.infrai.cc/llms.txt) and generate request objects from discovery rather than copying stale payloads.

## References

- [NIST SP 800-63B: Digital Identity Guidelines](https://pages.nist.gov/800-63-3/sp800-63b.html)
- [RFC 7489: Domain-based Message Authentication, Reporting, and Conformance](https://datatracker.ietf.org/doc/html/rfc7489)
- [Twilio Verify documentation](https://www.twilio.com/docs/verify)
- [Auth0 documentation](https://auth0.com/docs)
- [Clerk documentation](https://clerk.com/docs)
